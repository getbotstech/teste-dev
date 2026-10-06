# Teste técnico - Getbots

A Getbots constrói atendimento por WhatsApp para marcas grandes: promoção com nota fiscal, cadastro de cupom, consulta de pedido, triagem de suporte. Por dentro, quase todo projeto é a mesma história: uma conversa, uma regra de negócio e uma API de terceiro que a gente não controla. O modelo é a parte fácil.

Este desafio tem a cara desse dia a dia. É pequeno de propósito: um agente, duas integrações, uma tabela de regras. O que nos interessa não é a quantidade de coisa que você constrói, e sim as decisões que toma pelo caminho e se sabe explicá-las.

## O problema

A **Doce Vida**, uma marca fictícia de alimentos, lançou a promoção **Compre e Ganhe**: quem compra produtos da marca nas redes participantes cadastra a nota fiscal pelo WhatsApp, ganha números da sorte para o sorteio e recebe um kit de brindes em casa.

Você vai construir o agente que atende essa promoção.

O cliente manda a **chave de acesso** da nota (44 dígitos, impressa no cupom) e o valor da compra. A chave já carrega o estado, o mês da emissão e o CNPJ da loja; para saber que loja é essa e se ela existe, há duas APIs públicas de CNPJ. Uma delas sabemos que cai com frequência; a outra só aceita 3 consultas por minuto. Para enviar o kit, o agente pede o CEP e consulta duas APIs públicas de endereço, que também caem com frequência, demoram e nem sempre concordam entre si.

E o modelo, se deixar, lê a chave errado, arredonda o valor, dá número da sorte para nota de outra promoção e completa o endereço de memória.

Do outro lado tem alguém que acabou de fazer uma compra para participar. Ela precisa saber quantos números ganhou, quando o kit chega ou, se algo não deu certo, exatamente por quê.

## O que o agente faz

Recebe mensagens em linguagem natural e conduz duas etapas.

**1. Cadastro da nota.** Coleta chave de acesso e valor, e responde quantos números da sorte a nota rendeu.

- Lê o estado, o mês de emissão e o CNPJ de dentro da chave.
- Descobre razão social e situação cadastral do CNPJ.
- Aplica [`data/regras.csv`](data/regras.csv) e [`data/redes-participantes.csv`](data/redes-participantes.csv): a loja precisa ser de uma rede participante (mesmo CNPJ raiz, qualquer filial) e estar ativa; a nota precisa ter sido emitida em um dos estados e dentro do período da promoção; cada R$ 50 vale 1 número, até o máximo por nota.
- Uma nota só pode ser cadastrada uma vez. Se a mesma pessoa manda a nota de novo, o agente avisa que ela já está cadastrada e repete quantos números rendeu. Se outra pessoa manda uma chave que já foi cadastrada, o agente recusa, diz que a nota já pertence a outro participante, não revela nada sobre quem cadastrou e registra a tentativa para a equipe da promoção olhar.

**2. Envio do kit.** Com a nota cadastrada, pede o CEP e responde se o kit chega e em quantos dias.

- Descobre cidade e estado a partir do CEP.
- Aplica [`data/entregas.csv`](data/entregas.csv): a linha da cidade vale mais que a linha do estado, e estado fora da tabela não tem cobertura.
- Sem cobertura, os números da sorte valem mesmo assim; o agente explica que o kit não pode ser enviado.

As tabelas vieram do cliente do jeito que ele exporta da planilha dele. A de [`data/ufs-ibge.csv`](data/ufs-ibge.csv) é nossa, para traduzir o código de estado da chave.

### A chave de acesso

44 dígitos, nesta ordem:


| Posição | Tamanho | Conteúdo                       |
| ------- | ------- | ------------------------------ |
| 1-2     | 2       | Código IBGE da UF              |
| 3-6     | 4       | Ano e mês da emissão (`AAMM`)  |
| 7-20    | 14      | CNPJ do emitente               |
| 21-22   | 2       | Modelo (`55` NF-e, `65` NFC-e) |
| 23-25   | 3       | Série                          |
| 26-34   | 9       | Número da nota                 |
| 35      | 1       | Tipo de emissão                |
| 36-43   | 8       | Código numérico                |
| 44      | 1       | Dígito verificador (módulo 11) |


Exemplo válido: `35261045543915000181550010001234561123456784` (SP, outubro de 2026, CNPJ 45.543.915/0001-81).

## APIs disponíveis

CNPJ:

- BrasilAPI: `https://brasilapi.com.br/api/cnpj/v1/{cnpj}`
- ReceitaWS: `https://receitaws.com.br/v1/cnpj/{cnpj}` (3 consultas por minuto no plano gratuito)

CEP:

- ViaCEP: `https://viacep.com.br/ws/{cep}/json/` (para CEP que não existe, responde `200` com `{ "erro": "true" }`)
- BrasilAPI: `https://brasilapi.com.br/api/cep/v1/{cep}`

## Requisitos

### Endpoint

`POST /chat`

```json
{ "conversationId": "<id>", "message": "quero cadastrar a nota 35261045543915000181550010001234561123456784, deu 180 reais" }
```

```json
{ "reply": "Nota do Carrefour cadastrada! R$ 180,00 rendeu 3 números da sorte. Para enviar o seu kit, qual é o seu CEP?" }
```

```json
{ "conversationId": "<id>", "message": "01001-000" }
```

```json
{ "reply": "Fechado! O kit vai para São Paulo/SP e chega em até 1 dia." }
```

`reply` é o único campo obrigatório. Pode devolver outros (o que a etapa decidiu, as ferramentas chamadas, um id para rastrear a conversa): a gente ignora o que não conhece, e para você isso é o que permite conferir o `eval.csv` por código em vez de ler resposta por resposta.

### Comportamento esperado

- A conversa tem memória: chave, valor e CEP podem vir em mensagens diferentes, em qualquer ordem.
- Se uma API falhar, a outra do mesmo tipo é consultada automaticamente.
- Quantidade de números, nome da loja, motivo de recusa, endereço e prazo nunca são inventados.

### Cenários que vamos rodar

Estão em [`data/eval.csv`](data/eval.csv), uma linha por turno:

- `cenario` e `turno` ordenam a conversa; `pessoa` diz quem fala (`A` e `B` viram `conversationId`s distintos, então um cenário pode envolver duas pessoas).
- `mensagem` é o que o cliente manda naquele turno.
- O resto é o que a resposta daquele turno precisa conter: `resultado` do cadastro (`cadastrada`, `recusada`, `pendente` ou `erro`), `numeros`, `motivo` da recusa, `entrega` e `prazo_dias` do kit, e o `comportamento` esperado. Célula vazia é "não se aplica neste turno".

Cada cenário usa uma chave própria e começa uma conversa nova. Vamos rodar esses e variações deles. Use o arquivo como quiser; ele é para você também.

## O que queremos ver

Nem tudo aqui precisa virar código. O que queremos é entender como você pensa; onde não implementar, conte no seu README o que faria. O teste é o mesmo para júnior e pleno: o que muda é a profundidade com que cada um responde.

1. **Integração** - Como você isolaria os quatro provedores do resto do sistema? O que faria com o limite de 3 por minuto se 50 clientes cadastrassem ao mesmo tempo? A mesma loja consultada duas vezes precisaria de duas chamadas? O que aconteceria se uma API demorasse 30 segundos?
2. **Fronteira entre código e modelo** - Quem deveria ler a chave de acesso? Quem deveria contar os números da sorte? Quem decidiria se tem cobertura? O que aconteceria se o modelo arredondasse R$ 149,90 para R$ 150?
3. **Quando dá errado** - Nota recusada e CEP sem cobertura precisam de um motivo que o cliente entenda. O que a ferramenta devolveria para o modelo, e o que o modelo devolveria para o cliente? Como você evitaria que ele inventasse um prazo com a API de CEP fora?
4. **Duplicidade** - O enunciado diz o que fazer quando a mesma pessoa repete a nota e quando outra pessoa manda a mesma chave. Onde ficaria o registro das notas cadastradas? O `conversationId` bastaria para dizer que é a mesma pessoa? O que aconteceria se duas pessoas mandassem a mesma chave no mesmo segundo?
5. **Como você sabe que funciona** - Um agente não é determinístico, e cada rodada contra APIs e modelo reais custa tempo e dinheiro. O que você testaria sem eles, e o que só faria sentido testar com eles? Se mudasse o prompt amanhã, como saberia que piorou? O [`data/eval.csv`](data/eval.csv) existe para isso; conte como você o usou ou como o usaria.
6. **Observabilidade** - Um cliente diz que a nota dele valia 4 números e o agente deu 3. Outro diz que o kit nunca chegou e ninguém sabe que CEP ele informou. Como você descobriria o que aconteceu?

## Stack

TypeScript, Node.js e Fastify. O resto é escolha sua: modelo, provedor, framework de agente ou nenhum, biblioteca de testes. Se não tiver uma chave de modelo, há camadas gratuitas (Google AI Studio, Groq) e modelos locais (Ollama). Use o que souber defender.

## O que não estamos avaliando

- Frontend
- Banco de dados
- Deploy
- Integração real com WhatsApp
- Multi-agente, RAG ou base vetorial
- Cobertura de testes de 100%

## O que entregar

Um repositório seu com o código e três arquivos na raiz:

- **`README.md`** - Como rodar o projeto e os testes, e as decisões que você tomou: o que considerou, o que descartou e por quê, o que ficou de fora.
- **`PROCESSO.md`** - Como você usou IA para construir isto: quais ferramentas, o que delegou, como conferiu o que foi gerado, onde a ferramenta errou ou onde você discordou dela. Se usou prompts, specs ou arquivos de regras, suba junto. Não usar IA também é uma resposta válida; basta dizer.
- **`AGENTS.md`** - O arquivo que um agente de código leria antes de mexer no seu projeto: como rodar, como testar, onde fica cada coisa, convenções e o que não fazer. Se você usou um durante o desenvolvimento, é esse mesmo.

## Como entregar

Clone este repositório e implemente a solução em um repositório seu, público ou privado. Envie um e-mail para [jackson@getbots.com.br](mailto:jackson@getbots.com.br) com o link do repositório e o assunto **Teste Dev - Getbots**. Se o repositório for privado, adicione o usuário `serafimqwe` como colaborador.

