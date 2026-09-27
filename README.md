# Laboratório — a mesma loja, duas arquiteturas

Estilos Arquiteturais V · Microsserviços

Nomes: João Lucas Soares

Vocês vão rodar a **mesma loja** de dois jeitos: como um **monolito** (um processo, um banco) e como **microsserviços** (quatro processos, um banco por serviço). As duas versões têm as mesmas rotas. Ninguém precisa programar — só um experimento pede para mudar um número no código.

Requisito: Python 3.8 ou superior. Nada para instalar. No macOS/Linux use `python3`.

| Versão | Como subir | Endereço |
| --- | --- | --- |
| Monolito | `python monolito.py` | http://localhost:8000 |
| Microsserviços | `python iniciar_microsservicos.py` | http://localhost:9000 |

O objetivo é descobrir, na prática, **onde os microsserviços ganham e onde eles cobram**.

| Versão | Endereço | Onde ficam os dados |
| --- | --- | --- |
| Monolito | http://localhost:8000 | `dados/monolito.json` |
| Microsserviços | http://localhost:9000 (Vitrine) | `dados/estoque.json` e `dados/pedidos.json` |
| Rotas nas duas | `/produto/1`, `/comprar/1`, `/relatorio`, `/bug/estoque` | |
| Derrubar um serviço | http://localhost:<porta>/desligar | Catálogo 9001 · Estoque 9002 · Pedidos 9003 |

**Faça:** Abra dois terminais na pasta `laboratorio`.

```bash
# terminal 1 — leva 10 s para subir
python monolito.py
```

```bash
# terminal 2 — sobe 4 processos
python iniciar_microsservicos.py
```

**Faça:** Abra duas abas no navegador, lado a lado:

- http://localhost:8000/produto/1
- http://localhost:9000/produto/1

**Deve aparecer:** O mesmo produto nas duas: Teclado mecânico, R$ 250, 5 em estoque.

### Experimento 1 · Deploy de uma promoção

O time de vendas quer 10% de desconto. Vocês vão "publicar" essa mudança nas duas versões e observar **o que sai do ar enquanto isso**.

**Faça — monolito**

a) Em `monolito.py`, mude `DESCONTO = 0` para `DESCONTO = 10`.

b) No terminal 1, `Ctrl+C` e rode `python monolito.py` de novo.

c) Durante os 10 s de subida, tente abrir <http://localhost:8000/relatorio>.

**Faça — microsserviços**

a) Em `catalogo.py`, mude `DESCONTO = 0` para `DESCONTO = 10`.

b) Abra <http://localhost:9001/desligar> para derrubar só o Catálogo.

c) Num terceiro terminal, rode `python catalogo.py`.

d) Durante os 4 s de subida, tente <http://localhost:9000/relatorio> e <http://localhost:9000/produto/1>.

**Deve aparecer:** Depois da subida, o preço é R$ 225 nas duas.

| | Monolito | Microsserviços |
| --- | --- | --- |
| Quanto tempo ficou fora do ar? | uns 10 s | uns 4 s (só o catálogo) |
| O que parou de funcionar? | tudo, nenhuma página abria | a página do produto (e a compra, que depende do catálogo) |
| O que continuou funcionando? | nada | o relatório, estoque e pedidos |

**Responda: Poder ou problema dos microsserviços? Por quê?** Poder. No monolito a gente mudou só o desconto e a loja inteira ficou fora do ar enquanto reiniciava. Nos microsserviços só o catálogo caiu e o resto continuou funcionando, então dá pra fazer deploy de uma parte sem derrubar tudo.

### Experimento 2 · Latência

**Faça:** Num terminal livre, rode:

```bash
python comparar.py
```

**Faça:** Abra de novo `/produto/1` nas duas abas e compare o campo `tempo_interno_ms`.

| | Monolito | Microsserviços |
| --- | ---: | ---: |
| Tempo médio por página (`comparar.py`) | 10.04 ms | 29.96 ms |
| `tempo_interno_ms` | 0.018 | 19.954 |

**Responda: Aqui tudo roda no mesmo computador. O que aconteceria com essa diferença se cada serviço estivesse numa máquina diferente?** A diferença ia ficar bem maior, porque cada chamada entre os serviços teria que passar pela rede de verdade. Nos microsserviços cada página já faz 2 chamadas internas, então quanto mais serviços envolvidos, mais lento fica.

### Experimento 3 · Consistência

Na demonstração, o professor derrubou o Estoque e a loja em microsserviços **continuou vendendo**. Agora vocês vão ver o preço disso.

**Faça:**

a) Abra <http://localhost:9000/relatorio> e anote os números.

b) Derrube só o Estoque: <http://localhost:9002/desligar>.

c) Compre duas vezes: <http://localhost:9000/comprar/1>.

d) Religue o Estoque: `python estoque.py`.

e) Abra <http://localhost:9000/relatorio> de novo.

**Deve aparecer:** Em (c): `"aviso": "Estoque fora do ar: pedido registrado SEM baixa de estoque"`. Em (e): `"pedidos_pendentes": 2` e `"consistente": false`.

| /relatorio (microsserviços) | Antes | Depois |
| --- | ---: | ---: |
| pedidos_confirmados | 0 | 0 |
| pedidos_pendentes | 0 | 2 |
| baixas_de_estoque | 0 | 0 |
| consistente | true | false |

**Faça:** Abra a pasta `dados/`. Compare `pedidos.json` e `estoque.json`: cada banco conta uma história diferente.

O `pedidos.json` tem 2 pedidos do teclado como pendentes, mas no `estoque.json` o teclado continua com 5 unidades e 0 baixas.

**Responda: No monolito, o bug do Estoque derrubou a loja inteira: nenhuma venda, mas nenhum dado errado. Nos microsserviços, a loja vendeu, mas os bancos discordam. Qual dos dois a loja prefere? Quem decide isso?** Na maioria dos casos a loja prefere continuar vendendo e depois corrigir os pedidos pendentes, porque perder venda costuma ser pior. Mas isso depende do negócio (por exemplo, se o produto tem pouco estoque pode ser melhor não vender). Quem decide é a loja/área de negócio, não os desenvolvedores.

### Experimento 4 · Operação

**Faça:** Faça uma compra em cada versão (`/comprar/2` nas duas) e olhe os terminais.

| Para uma compra… | Monolito | Microsserviços |
| --- | --- | --- |
| Quantos processos estão rodando? | 1 | 4 |
| Quantas portas? | 1 | 4 |
| Quantos arquivos de banco? | 1 | 2 |
| Quantas linhas de log apareceram? | 1 | 4 |
| Em quantos serviços? | 1 | 4 |

**Responda: Se a compra desse errado, onde vocês procurariam o erro em cada versão?** No monolito é só olhar o log de um terminal, já que tudo está no mesmo processo. Nos microsserviços teria que olhar o log de cada serviço (vitrine, pedidos, catálogo e estoque) até achar em qual deles deu o erro, o que dá mais trabalho.

## Placar da dupla

| Experimento | Quem saiu melhor? | Para microsserviços, é poder ou problema? |
| --- | --- | --- |
| Bug fatal (demonstração) | microsserviços | poder |
| 1 · Deploy | microsserviços | poder |
| 2 · Latência | monolito | problema |
| 3 · Consistência | monolito | problema |
| 4 · Operação | monolito | problema |

**Para fechar: Uma loja com 3 desenvolvedores deveria usar qual das duas versões? E uma com 300? Usem o placar como argumento.** Com 3 desenvolvedores o monolito é melhor: é mais simples de rodar e de achar erro, é mais rápido e não tem problema de dados inconsistentes. Com poucas pessoas, o deploy independente não faz tanta diferença. Com 300 desenvolvedores os microsserviços compensam, porque cada time pode cuidar do seu serviço e fazer deploy sem derrubar a loja toda, e um bug em uma parte não para tudo. Aí vale a pena lidar com a latência e a operação mais complicada.

Para recomeçar do zero: desliguem tudo (`Ctrl+C` nos terminais) e rodem `python resetar.py`.
