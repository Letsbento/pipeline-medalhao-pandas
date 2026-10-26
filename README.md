# Pipeline Medalhão — MercadoNordeste (local + pandas)

Pipeline de dados em camadas (landing → Bronze → Silver → Gold) rodando localmente com **pandas** e **MinIO** (object storage S3) em Docker.

**Autora:** Letícia Bento

## Como rodar

```powershell
py -3.12 -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt

copy docker\.env.example docker\.env

cd docker
docker compose up -d
```

Depois, rode os notebooks em ordem: `notebooks/01_bronze_local_pandas.ipynb` → `02_silver_local_pandas.ipynb` → `03_gold_local_pandas.ipynb` (Parte A). Os dados de entrada ficam em `data/lote1/`.

## Estrutura

```
├── data/lote1/          clientes.csv, produtos.csv, pedidos.jsonl, avaliacoes.csv
├── docker/
│   ├── docker-compose.yml
│   └── .env.example     modelo das variáveis (sem senhas reais)
├── notebooks/           01_bronze, 02_silver, 03_gold (executados, com saídas)
└── requirements.txt
```

---

## Parte 2 · Evidências

As saídas estão nos próprios notebooks executados, em `notebooks/`.

| Evidência | Onde |
|---|---|
| Contagem de linhas por tabela e reexecução do mesmo lote sem duplicar (idempotência) | `01_bronze`, Passos 2 e 5 |
| Resumo da construção, quarentena por origem e reconciliação (entrada = Silver + quarentena) | `02_silver`, Passo 2, Reconciliação e "O que foi para a quarentena?" |
| Relatório de DQ e tabela `dq_resultados` | `02_silver`, Data Quality |
| Erro simulado (pedidos cancelados apagados) e recuperação da versão anterior | `02_silver`, Histórico de versões |
| Tabela de gates com todos os itens OK, as três tabelas publicadas e o contrato de dados | `03_gold`, Passos 1, 4 e 5 |

---

## Parte 3 · Análise

### Qual regra da Silver mandou mais registros para a quarentena, e por quê?

A regra de produto inexistente nos itens de pedido, com 747 itens. As outras origens tiveram bem menos: 40 pedidos de clientes que não existem (`pedidos_cliente_orfao`), 25 avaliações com nota fora de 1 a 5 e 3 produtos com preço ou custo inválido.

O motivo é um efeito em cascata. Os 3 produtos foram para a quarentena por terem preço negativo (por exemplo, "Jogo de Cama Max", com `-418.73`). Como eles saíram da Silver, todos os itens de pedido que os citavam passaram a apontar para um produto que "não existe". Um erro em 3 linhas do cadastro virou 747 itens rejeitados.

### Se a reconciliação não fechasse, o que eu investigaria primeiro?

Isso aconteceu de verdade na Gold: o gate `pedidos Gold = pedidos válidos Silver` falhou, com 22.473 pedidos válidos na Silver contra 22.286 em `vendas_diarias`.

O caminho que segui:
1. Ver qual número não fechava. A receita batia; só a contagem de pedidos é que não.
2. Pegar a diferença de IDs entre os dois lados. Eram 187 pedidos.
3. Procurar esses IDs na quarentena. Eram pedidos com status válido, mas com todos os itens rejeitados (itens dos produtos de preço negativo). Sem item válido, não há receita, e o pedido não aparece em `vendas_diarias`.
4. Corrigir a regra, não o dado. A reconciliação passou a contar só os pedidos válidos que têm pelo menos um item válido.

Ou seja, a primeira coisa a investigar é a quarentena e as regras que tiram registros do caminho, antes de suspeitar do cálculo.

### Que dado pessoal existe e como foi tratado?

| Dado | Na Silver | Na Gold |
|---|---|---|
| `nome` | completo | mascarado (`Ana Souza` → `A** S****`) |
| `email` | padronizado; os inválidos viram NULL | removido (minimização: não é necessário para a pergunta de negócio) |
| `cliente_id` | sim | sim (pseudônimo: liga à pessoa, mas não a identifica sozinho) |
| `comentario` (texto livre) | como veio | não entra nas tabelas de negócio |

Além disso, um gate da Gold bloqueia a publicação se aparecer qualquer coluna `nome`, `email`, `cpf` ou `telefone`. O acesso também é restrito: as senhas ficam no `docker/.env`, fora do código e fora do repositório (`.gitignore`).

Ponto de atenção: a Silver ainda guarda nome e e-mail completos, e todas as camadas do MinIO usam a mesma credencial de administração. Em produção, cada camada deveria ter credenciais e permissões próprias.

### O que eu mudaria se o volume fosse 1.000 vezes maior?

- Trocar o pandas por um motor distribuído (Spark). O pandas carrega tudo na memória: 30 mil pedidos viram 30 milhões.
