# Plano técnico — Projeto “Relatório de Entradas”

## Visão geral
O sistema atual (PHP + MySQL) possui módulos de **Cadastro de Empenhos** e **Cadastro de Bens**. O projeto “Relatório de Entradas” adiciona três funcionalidades principais:

1. **Relatório de Entradas (tela principal)**: lista consolidada de bens incorporados, integrando empenhos e bens.
2. **Valores Liquidados (mensal)**: conferência entre o relatório financeiro externo e os dados internos.
3. **Fechamento Anual**: resumo anual das movimentações com totais mensais e por categoria contábil.

## 1) Relatório de Entradas (tela principal)

### Objetivo
Apresentar uma tabela consolidada por **pasta** com os campos: Pasta, Conta Contábil, Equipamento, Quantidade, Fornecedor, N.F., Valor Unit., Valor Total, Data, Secretaria, Nº Patrimônio (intervalo), Nº Empenho e Data Empenho.

### Integração de dados (Empenhos x Bens)
- **Pasta**: `Cadastro_Bens.pasta` (lote de entrada).
- **Conta Contábil**: `Cadastro_Bens.categoria` (derivada de `empenhos.conta_categoria`).
- **Equipamento**: `Cadastro_Bens.equipamento_material`.
- **Quantidade**: `COUNT(*)` por pasta.
- **Fornecedor**: `empenhos.fornecedor`.
- **N.F.**: `Cadastro_Bens.n_nota`.
- **Valor Unit.**: `Cadastro_Bens.valor_do_bem`.
- **Valor Total**: `SUM(valor_do_bem)` por pasta.
- **Data**: `data_recebimento_nota` (preferencial), senão `data_nf` ou `data`.
- **Secretaria**: `Cadastro_Bens.secretaria`.
- **Nº Patrimônio**: `MIN(sequencia)` e `MAX(sequencia)`.
- **Nº Empenho**: `empenhos.empenho` + `empenhos.ano_empenho`.
- **Data Empenho**: `empenhos.dt_emp`.

### Query base (sugestão)
```sql
SELECT
  b.pasta,
  e.conta_categoria,
  b.equipamento_material,
  COUNT(b.id) AS quantidade,
  e.fornecedor,
  b.n_nota,
  b.valor_do_bem,
  SUM(b.valor_do_bem) AS valor_total,
  COALESCE(b.data_recebimento_nota, b.data_nf, b.data) AS data_entrada,
  b.secretaria,
  MIN(b.sequencia) AS patrimonio_inicial,
  MAX(b.sequencia) AS patrimonio_final,
  e.empenho,
  e.ano_empenho,
  e.dt_emp AS data_empenho
FROM Cadastro_Bens b
JOIN empenhos e
  ON e.empenho = b.empenho
GROUP BY b.pasta
ORDER BY b.pasta ASC;
```

> **Observação sobre o ano do empenho**: `Cadastro_Bens` armazena apenas o número do empenho. Caso haja números repetidos entre anos, a associação deve ser ajustada (ex.: armazenar `ano_empenho` no cadastro de bens ou usar heurística com a data do bem).

### Interface e exportação PDF
- **Página**: `relatorio_entradas.php`, seguindo layout do sistema.
- **Filtro**: opcional por fornecedor, conta contábil, período.
- **PDF**: botão “Exportar PDF” com `domPDF` (A4 paisagem).

### Etapas
1. Criar página e permissões.
2. Implementar query agregada.
3. Renderizar tabela e formatação de valores/datas.
4. Exportar em PDF.
5. Adicionar ao menu.

## 2) Valores Liquidados (conferência mensal)

### Objetivo
Cruzar o PDF de **Despesa Liquidada** com os dados internos, apontando divergências por empenho no mês.

### Importação do PDF
- **Upload**: arquivo `.pdf` (formulário na interface).
- **Parser**: `smalot/pdfparser` ou `pdftotext` (Poppler).
- **Extração**: regex para linhas que começam com data e terminam com valor.
- **Acumulação**: somar valores por empenho.

### Tabela sugerida
```sql
CREATE TABLE despesas_liquidadas (
  id INT AUTO_INCREMENT PRIMARY KEY,
  ano SMALLINT,
  mes TINYINT,
  empenho INT,
  valor_liquidado DECIMAL(15,2)
);
```

### Cruzamento e exibição
Para cada empenho do PDF:
- Localizar empenho interno por número e ano.
- Somar valores de bens no mês (`Cadastro_Bens`) e comparar com o valor liquidado do mês.
- Exibir **Diferença** e **Status** (OK / Divergente / Não cadastrado).

### Etapas
1. Upload e parsing do PDF.
2. Persistir valores por mês em `despesas_liquidadas`.
3. Cruzar com bens do mês.
4. Exibir tabela e exportar PDF.
5. Tratar erros de parsing e dados inconsistentes.

## 3) Fechamento anual

### Objetivo
Gerar relatório anual com totais mensais por categoria (A–I) e totais por conta contábil.

### Categorias sugeridas
- **A**: liquidados do exercício.
- **B**: restos a pagar reempenhados.
- **C**: doações.
- **D**: estorno de entrada.
- **E**: estorno de saída.
- **F**: reclassificação.
- **G**: outras movimentações (se aplicável).
- **H**: prioridades não pagas.
- **I**: estorno de doação.

### Estratégia de cálculo (sugerida)
- Basear-se em **bens incorporados** no ano para A/B/C.
- Usar `empenhos` (condição/status) para classificar reempenho e prioridades.
- Tratar estornos via status dos bens (recomendado adicionar “ESTORNADO”).

### Totais por conta contábil
```sql
SELECT categoria, SUM(valor_do_bem) AS total_valor
FROM Cadastro_Bens
WHERE YEAR(data) = :ano
  AND (status IS NULL OR status != 'ESTORNADO')
GROUP BY categoria;
```

### Saída
- Página `fechamento_anual.php` com escolha do ano.
- PDF com layout conforme modelo (A4 paisagem).

## 4) Segurança e controle de acesso
- Autenticação por sessão.
- Permissões: `contabilidade_empenho` ou equivalentes.
- Upload de PDF: validação de MIME/tamanho e SQL preparado.
- Sanitização de saída (HTML/PDF).

## 5) Automação (opcional)
- **Importação mensal automática** via cron (se PDF disponível em diretório padrão).
- **Geração anual automática** no início do ano (salvar PDF em diretório protegido).
- **Backup** regular da tabela `despesas_liquidadas` e PDFs.

## 6) Próximos passos técnicos
1. Implementar o Relatório de Entradas e exportação PDF.
2. Criar o fluxo mensal (upload + parsing + conferência).
3. Construir o fechamento anual (cálculo e PDF).
4. Validar com dados reais e ajustar regras de classificação.

