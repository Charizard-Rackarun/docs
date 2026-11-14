# VitrineIA — Resumo para Avaliadores

**Em uma frase:** plataforma que transforma pesquisas rápidas e anônimas com visitantes de um shopping em inteligência comercial para os lojistas, e devolve valor ao visitante em pontos e cupons.

*Protótipo funcional · Frontend Next.js + Backend FastAPI · 14/11/2026*

---

## 1. O problema

Lojistas de shopping vendem sem saber **o que o visitante procura** nem **quais lojas vizinhas atraem o mesmo público**. Pesquisas tradicionais têm baixa adesão e exigem coletar dados pessoais, o que esbarra na LGPD.

## 2. A solução

1. O lojista publica um **questionário curto** e compartilha um link/QR code.
2. O visitante responde **sem cadastro** e ganha **pontos**.
3. Os pontos viram **cupons** das lojas, validados no balcão com um código único.
4. As respostas, sempre **agregadas e anônimas**, alimentam **segmentos de público** e **oportunidades de venda cruzada** (ex.: "quem gosta de Moda também se interessa por Beleza"), com recomendação em linguagem natural gerada por IA.

## 3. Diferenciais

| Diferencial | Como foi resolvido |
|---|---|
| **Privacidade por construção (LGPD)** | Visitante sem login e sem dado pessoal; só o hash de um token aleatório é guardado |
| **Anti-reidentificação** | Grupos com menos de 5 respostas nunca são exibidos (k-anonimato configurável) |
| **IA com responsabilidade** | Só números agregados vão ao provedor; sem chave ou em caso de falha, o sistema usa texto gerado por regras e continua funcionando |
| **Análise explicável** | Regras de associação com suporte, confiança e lift, classificadas em prioridade Alta/Média/Baixa |
| **Engajamento** | Pontos, níveis de fidelidade e o evento temático **Sexta 13** (pontos em dobro e cupons secretos) |
| **Robustez** | Resgate com bloqueio de linha no banco (sem resgate duplo ou estoque negativo) e restrição de uma resposta por visitante |
| **Multi-lojista seguro** | Cada lojista só enxerga os próprios dados |

## 4. O que está implementado

**Área do lojista (com login)**
- Cadastro e login (JWT, senha com Argon2)
- Painel: faturamento, vendas, ticket médio, consumidores ativos, série mensal, vendas por segmento e atividade recente
- Questionários: criação, pausa/reativação, link para divulgação e resultados por pergunta
- Consumidores (segmentos), Oportunidades (geração com IA), Campanhas, Cupons, Registro de vendas
- Validação de resgate no balcão

**Área do visitante (sem login)**
- Resposta a questionário com feedback de pontos (e bônus Sexta 13)
- Carteira de pontos, nível e resgate de cupons com código

## 5. Arquitetura em resumo

```
Next.js (App Router, Tailwind, Recharts)  ──JSON──►  FastAPI (async) ──► Banco relacional
        │                                                  │
 localStorage: JWT do lojista                              └─► API de IA (opcional, só dados agregados)
 e token anônimo do visitante
```

- Contratos tipados nos dois lados (Pydantic ↔ TypeScript, em camelCase).
- Lógica de análise isolada em serviços (`analytics`, `dashboard`, `ia`, `sexta13`), testável de forma independente.
- Detalhes completos em **`documentacao-tecnica.md`**.

## 6. Roteiro sugerido de demonstração (≈5 min)

1. **Cadastro** de uma loja em `/registro`.
2. **Criar questionário** com "Usar modelo de interesses" e publicar.
3. **Criar cupom** em `/cupons` (validade futura).
4. **Responder como visitante** em `/responder/1` (outra janela ou perfil) e ver os pontos ganhos.
5. **Resgatar** o cupom em `/rewards` e copiar o código.
6. **Validar** o código em `/cupons` como lojista.
7. Com 5 ou mais respostas de visitantes distintos, mostrar **Consumidores**, **Resultados** e **Oportunidades** (botão "Gerar análise com IA").

> Dica: use perfis/janelas anônimas diferentes para simular vários visitantes; cada navegador é um visitante.

## 7. Limitações assumidas (honestidade técnica)

- **Pontos ficam no navegador do visitante.** É o preço de não pedir dados pessoais: limpar o navegador ou trocar de aparelho zera o saldo. Evolução natural: login opcional com consentimento.
- **Vitrine de cupons filtrada.** O visitante só vê cupons ativos, dentro da validade, com estoque e, no caso das ofertas secretas, apenas em Sextas 13. O painel do lojista ainda não mostra o motivo quando um cupom não aparece (melhoria planejada).
- O visitante ainda **não vê o custo em pontos** de cada cupom (campo previsto no backend, mas não exposto pela API).
- Não há listagem pública de questionários: o acesso é por link/QR code.
- Protótipo sem testes automatizados, sem *rate limiting* e com JWT em `localStorage`, itens listados com sugestões na documentação técnica.

## 8. Próximos passos

Login opcional para visitantes · exposição do custo em pontos · status de visibilidade dos cupons no painel · testes automatizados das regras de análise e de resgate · *rate limiting* e anti-bot · taxonomia compartilhada de categorias para sugerir lojas parceiras com mais precisão.

---

**Documentos relacionados:** `documentacao-tecnica.md`