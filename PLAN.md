# Projeto: pesquisa offline rápida para Android (por esporte)

Núcleo em Rust + llama.cpp via FFI. Zig opcional para o build. Sem pretensão de bounty: o objetivo é medir e ganhar velocidade.

## Dispositivo-alvo: o celular mais básico possível
Objetivo de acessibilidade: LLM offline para todas as pessoas, não só para quem tem aparelho top de linha. O piso é uma **classe** de aparelho (a ser confirmada com medição em aparelho real, não assumida):
- Android 10+ (arm64-v8a), 3–4 GB de RAM (sobra útil para o app: ~1,5–2 GB), CPU de entrada (ex.: Cortex-A53/A55, Snapdragon 4xx/6xx, Helio G-series), 32–64 GB de armazenamento, sem GPU/NPU utilizável.
- Consequência: modelo de geração ≤ ~1B parâmetros quantizado (candidatos a medir: Qwen2.5/3 0,5–0,6B, Gemma 3 1B, Llama 3.2 1B), pack de conhecimento de poucos GB, mmap agressivo.
- MoE com streaming (Fase 5) e aceleradores (Fase 6) passam a ser opcionais para aparelhos melhores; o caminho principal é **CPU puro**.
- Nível "premium" (8–12 GB) continua como segundo perfil de teste.

## Metas mensuráveis
Alvos provisórios; serão recalibrados depois da primeira medição no aparelho básico.
| Métrica | Alvo (básico) | Alvo (premium 8–12 GB) |
|---|---|---|
| Primeiro resultado (passagens) | < 1 s | < 1 s |
| Primeiro token da síntese | < 10 s | < 5 s |
| Geração | > 5 tok/s | > 10 tok/s |
| Pico de RAM do app | < 1,5 GB | < 4 GB |
| Tamanho total (modelo + índice) | < 8 GB | < 50 GB |
| Rede | zero (APK sem permissão INTERNET) | zero |

Sem modelo, a busca por passagens já deve entregar valor: é o modo que funciona até em aparelho sem RAM para o LLM.

Referência (Bivouac, auto-medido): busca 6 s, prefill 12,7 tok/s, geração 3,2 tok/s, resposta ~110 s.

## Princípios
1. Medir antes de otimizar. Nada entra sem número no harness.
2. Não reescrever kernels de matmul. Reaproveitar o ggml/llama.cpp.
3. Rust/Zig onde ajudam: índice, busca, compressão de contexto, especulação, build.
4. Emulador serve para corretude e UI. **Velocidade só vale em celular real** (emulador x86 não tem NEON/i8mm, a banda de memória é do PC).

## Fases

### Fase 0 — Ambiente (PC)
- Rust estável + alvos `aarch64-linux-android` e `x86_64-linux-android`, `cargo-ndk`.
- Android SDK/NDK, AVD x86_64 com KVM (ou Android Studio emulator), `adb`.
- llama.cpp como submódulo fixado em um commit; build com NDK.
- Opcional: Zig (`zig cc`) como toolchain alternativa; só se simplificar o build.
- Critério: "hello world" Rust+JNI rodando no emulador.

### Fase 1 — Harness de benchmark (primeiro de tudo)
- Conjunto fixo de ~60 perguntas (fatos, explicação, comparação, síntese, viagem, raciocínio), versionado no repo.
- Métricas por pergunta: t_busca, t_primeiro_token, tok/s geração, tok/s prefill, RAM de pico, tamanho do prompt.
- Avaliação de qualidade: rubrica cega (modelo-juiz local no PC + checagem manual por amostragem). Registrar a variância.
- Roda em PC (CLI) e no celular (via adb).
- Critério: relatório reprodutível em Markdown/JSON.

### Fase 2 — Corpus e índice em Rust
- Fonte inicial: subconjunto pequeno da Wikipedia (ex.: top ~100k artigos por pageviews) para iterar rápido; depois escalar.
- Pipeline: dump -> texto limpo -> chunks -> resumos/trechos comprimidos -> índice.
- Índice próprio: BM25 com FST/mmap (avaliar `tantivy` primeiro; formato próprio só se medir ganho). Meta < 50 ms por consulta.
- Reranker pequeno opcional (cross-encoder quantizado), medir custo/benefício.
- Wikivoyage + dados de lugares numa segunda etapa.
- Critério: recall@6 medido no conjunto de perguntas; latência no harness.

### Fase 3 — Pipeline de resposta rápido
- UI mostra passagens/trechos extraídos imediatamente; síntese em streaming depois.
- Compressão de contexto: enviar ao LLM só sentenças relevantes (alvo ~500 tokens de prefill).
- Cache de KV do prefixo fixo (instruções) e de conversa.
- Respostas com citação numerada; checagem simples de afirmações contra as fontes (fase posterior).
- Critério: primeiro token < 5 s no celular alvo.

### Fase 4 — Decodificação especulativa
- Lookup de n-gramas nas passagens recuperadas como rascunho (prompt lookup decoding), verificação pelo modelo.
- Medir aceitação e ganho de tok/s; sem perda de qualidade (lossless).
- Implementar o buscador de rascunhos em Rust.

### Fase 5 — MoE com streaming de experts
- Candidatos: Qwen3-30B-A3B (verificar disponibilidade/quantização), outros MoE de poucos ativos.
- Estudar BigMoeOnEdge e PRs do llama.cpp (#25294, #31469) antes de escrever algo próprio.
- Técnicas: cache LRU de experts, I/O direto em paralelo, prefetch, pesos densos em memória anônima.
- Critério: > 5 tok/s com qualidade claramente acima do 4B denso (medir no harness).

### Fase 6 — Aceleradores
- Testar Vulkan/OpenCL (Adreno/Mali) e Hexagon NPU no llama.cpp no celular real.
- Verificar comportamento no GrapheneOS (sem Play Services).
- Ficar só se o ganho for medido e estável.

### Fase 7 — Empacotamento
- APK `clean` sem INTERNET; gestão de packs/modelos por arquivo local (adb push ou importação).
- README com reprodução do índice, modelos, checksums e licenças.

## Riscos e decisões em aberto
- Qualidade vs. velocidade: o 4B denso já cumpre qualidade modesta; MoE é a aposta para subir sem perder velocidade.
- Idea "n-gram/memória condicional" do Vitalik: não há checkpoint conhecido utilizável; só vale se aparecer modelo aberto. Verificar na pesquisa antes.
- Licenças dos datasets e modelos (Wikipedia CC BY-SA, licenças dos pesos).
- Rede/disco da sessão podem limitar o download da Wikipedia completa; começar com subconjunto.
- Dados sobre concorrentes (Bivouac, Ostraka, BigMoeOnEdge) são auto-reportados e não foram reproduzidos.

## Perguntas que a nova sessão deve resolver cedo
1. ~~Modelo do celular~~ Decidido: o piso é um aparelho básico (ver acima). Falta escolher um aparelho físico de referência (RAM, SoC, armazenamento livre) para as medições.
2. ~~PC~~ Mac M1, 8 GB, macOS; emulador via AVD arm64 (sem KVM).
3. ~~Código MIT~~ Autorizado: pode ser usado como base ou comparação, com atribuição (arquivo `NOTICE`/README).
4. Repositório: github.com/developerfred/offline-research, licença MIT.

