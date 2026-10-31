Estou começando um projeto por esporte: um app Android de pesquisa totalmente offline e muito rápido (núcleo em Rust, llama.cpp via FFI, Zig opcional no build). Leia o PLAN.md ao lado antes de tudo.

Contexto:
- Inspirado no desafio do Vitalik (poidh bounty #31), mas NÃO estamos competindo pelo prêmio.
- Gargalos medidos por projetos parecidos (Bivouac): prefill ~12 tok/s, geração ~3 tok/s, resposta ~110 s. A linguagem não acelera matmul; o ganho vem de prompt menor, mostrar a busca antes do LLM, decodificação especulativa por lookup no prompt, MoE com streaming de experts e aceleradores.
- Emulador serve para corretude/UI; velocidade só vale em celular real.

Comece pela Fase 0 e Fase 1 (ambiente + harness de benchmark). Antes de escrever código, me pergunte o modelo do celular e o SO/hardware do PC. Não invente números: meça.

