# offline-research

Pesquisa offline rápida para Android, pensada para rodar nos celulares mais básicos, para que um LLM local seja acessível a todas as pessoas.

Núcleo em Rust, llama.cpp via FFI, Zig opcional no build. Zero rede: o APK não pede permissão `INTERNET`.

**Status:** início do projeto (Fase 0/1). Ainda não há código nem medições. Veja [PLAN.md](PLAN.md) para metas, fases e riscos.

## Princípios
- Medir antes de otimizar; nenhum número entra sem passar pelo harness.
- Não reescrever kernels de matmul; reaproveitar ggml/llama.cpp.
- Velocidade só vale em celular real; emulador serve para corretude e UI.

## Licença
[MIT](LICENSE). Código de terceiros, quando usado, é listado em `NOTICE` com atribuição.
