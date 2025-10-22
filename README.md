# 🎯 Objetivo do Módulo Finanças

Gerenciar tudo que envolve Fluxo de Caixa, Contas a Pagar, Contas a Receber, Conciliação Bancária e integração com vendas/compra para automático registro financeiro.

Fluxos principais:

Quando uma venda é finalizada → gera conta a receber (fatura).

Quando uma compra/fornecedor é registrada → gera conta a pagar.

Importar extratos bancários (CSV/CNAB) → ✓ concilia pagamentos/recebimentos com contas.

Painel com saldo diário, por conta bancária e por centro de custo.

🗂 1) Modelo de dados (SQL)

Adicione ao schema.sql:

-- contas_financeiras: armazena contas a pagar e a receber
CREATE TABLE contas_financeiras (
  id INT AUTO_INCREMENT PRIMARY KEY,
  tipo ENUM('receber','pagar') NOT NULL,
  descricao VARCHAR(255),
  valor DECIMAL(12,2) NOT NULL,
  data_vencimento DATE,
  data_pagamento DATE,
  status ENUM('aberto','pago','parcial','vencido') DEFAULT 'aberto',
  origem_tipo VARCHAR(50), -- 'venda','compra','manual'
  origem_id INT, -- id da venda/compra para link
  conta_bancaria_id INT, -- FK para contas bancarias
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- contas_bancarias: opcional, para consolidação por conta (ex: Conta Corrente, Caixa)
CREATE TABLE contas_bancarias (
  id INT AUTO_INCREMENT PRIMARY KEY,
  nome VARCHAR(100) NOT NULL,
  banco VARCHAR(100),
  agencia VARCHAR(20),
  conta VARCHAR(50),
  saldo_inicial DECIMAL(12,2) DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- movimentos_bancarios: extratos importados ou lançados automaticamente
CREATE TABLE movimentos_bancarios (
  id INT AUTO_INCREMENT PRIMARY KEY,
  conta_bancaria_id INT,
  data_movimento DATE,
  descricao VARCHAR(255),
  valor DECIMAL(12,2),
  tipo ENUM('credito','debito'),
  arquivo_origem VARCHAR(255), -- nome do arquivo de importação
  conciliado_com_conta_id INT, -- FK contas_financeiras.id, se conciliado
  conciliado_em DATETIME,
  referencia_external VARCHAR(100),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

🧭 2) Backend — exemplos (Node.js + Sequelize style)

Crie pasta: backend/src/modules/financas/

financas.model.js (Sequelize: contas_financeiras)
import db from '../../database/connection.js';
import { DataTypes } from 'sequelize';

const ContaFinanceira = db.define('ContaFinanceira', {
  id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
  tipo: { type: DataTypes.ENUM('receber','pagar'), allowNull: false },
  descricao: { type: DataTypes.STRING(255) },
  valor: { type: DataTypes.DECIMAL(12,2), allowNull: false },
  data_vencimento: { type: DataTypes.DATEONLY },
  data_pagamento: { type: DataTypes.DATEONLY },
  status: { type: DataTypes.ENUM('aberto','pago','parcial','vencido'), defaultValue: 'aberto' },
  origem_tipo: { type: DataTypes.STRING(50) },
  origem_id: { type: DataTypes.INTEGER },
  conta_bancaria_id: { type: DataTypes.INTEGER },
}, {
  tableName: 'contas_financeiras',
  timestamps: true,
  createdAt: 'created_at',
  updatedAt: 'updated_at'
});

export default ContaFinanceira;

banco.model.js
import db from '../../database/connection.js';
import { DataTypes } from 'sequelize';

const ContaBancaria = db.define('ContaBancaria', {
  id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
  nome: { type: DataTypes.STRING(100), allowNull: false },
  banco: { type: DataTypes.STRING(100) },
  agencia: { type: DataTypes.STRING(20) },
  conta: { type: DataTypes.STRING(50) },
  saldo_inicial: { type: DataTypes.DECIMAL(12,2), defaultValue: 0 }
}, { tableName: 'contas_bancarias', timestamps: true });

export default ContaBancaria;

movimento.model.js
import db from '../../database/connection.js';
import { DataTypes } from 'sequelize';

const MovimentoBancario = db.define('MovimentoBancario', {
  id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
  conta_bancaria_id: { type: DataTypes.INTEGER },
  data_movimento: { type: DataTypes.DATEONLY },
  descricao: { type: DataTypes.STRING(255) },
  valor: { type: DataTypes.DECIMAL(12,2) },
  tipo: { type: DataTypes.ENUM('credito','debito') },
  arquivo_origem: { type: DataTypes.STRING(255) },
  conciliado_com_conta_id: { type: DataTypes.INTEGER },
  conciliado_em: { type: DataTypes.DATE }
}, { tableName: 'movimentos_bancarios', timestamps: true });

export default MovimentoBancario;

financas.controller.js (exemplo com operações essenciais)
import ContaFinanceira from './financas.model.js';
import MovimentoBancario from './movimento.model.js';
import ContaBancaria from './banco.model.js';

// criar conta (pagar/receber)
export async function criarConta(req, res) {
  try {
    const c = await ContaFinanceira.create(req.body);
    res.status(201).json(c);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
}

// listar contas (com filtros)
export async function listarContas(req, res) {
  const { tipo, status, periodo_inicio, periodo_fim } = req.query;
  const where = {};
  if (tipo) where.tipo = tipo;
  if (status) where.status = status;
  if (periodo_inicio && periodo_fim) {
    where.data_vencimento = { [Op.between]: [periodo_inicio, periodo_fim] };
  }
  const contas = await ContaFinanceira.findAll({ where, order: [['data_vencimento','ASC']] });
  res.json(contas);
}

// marcar como pago (conciliar pagamento manual)
export async function marcarPago(req, res) {
  const { id } = req.params;
  const { data_pagamento, conta_bancaria_id, movimento_bancario_id } = req.body;
  const conta = await ContaFinanceira.findByPk(id);
  if (!conta) return res.status(404).json({ error: 'Conta não encontrada' });

  conta.data_pagamento = data_pagamento || new Date();
  conta.status = 'pago';
  conta.conta_bancaria_id = conta_bancaria_id || null;
  await conta.save();

  // opcional: marcar movimento bancario como conciliado
  if (movimento_bancario_id) {
    const mov = await MovimentoBancario.findByPk(movimento_bancario_id);
    if (mov) {
      mov.conciliado_com_conta_id = conta.id;
      mov.conciliado_em = new Date();
      await mov.save();
    }
  }

  res.json(conta);
}

// importar extrato (CSV simples) -> criar movimentos bancarios
export async function importarExtratoCsv(req, res) {
  // supondo upload do arquivo em req.file (use multer)
  const fileBuffer = req.file.buffer.toString('utf8');
  // parser CSV simples (ex.: data,descrição,valor,tipo)
  const lines = fileBuffer.split('\n').filter(Boolean);
  const registros = [];
  for (let line of lines) {
    const [data, descricao, valor, tipo] = line.split(',');
    const mov = await MovimentoBancario.create({
      conta_bancaria_id: req.body.conta_bancaria_id,
      data_movimento: data,
      descricao: descricao,
      valor: parseFloat(valor),
      tipo: tipo === 'C' ? 'credito' : 'debito',
      arquivo_origem: req.file.originalname
    });
    registros.push(mov);
  }
  res.json({ imported: registros.length });
}

// listar movimentos
export async function listarMovimentos(req, res) {
  const movimentos = await MovimentoBancario.findAll({ order: [['data_movimento','DESC']] });
  res.json(movimentos);
}


Observação: para parser CSV robusto use csv-parse ou papaparse. Para upload use multer.

financas.routes.js
import express from 'express';
import {
  criarConta, listarContas, marcarPago,
  importarExtratoCsv, listarMovimentos
} from './financas.controller.js';
import multer from 'multer';
const upload = multer();

const router = express.Router();

router.post('/financas/contas', criarConta);
router.get('/financas/contas', listarContas);
router.post('/financas/contas/:id/pagar', marcarPago);

router.post('/financas/extrato/import', upload.single('file'), importarExtratoCsv);
router.get('/financas/movimentos', listarMovimentos);

export default router;


Adicione ao main.ts:

import financasRoutes from './modules/financas/financas.routes.js';
app.use('/api', financasRoutes);

🔌 3) Endpoints REST essenciais (resumo)

GET /api/financas/contas — listar contas (filtro por tipo/status)

POST /api/financas/contas — criar conta (receber/pagar)

POST /api/financas/contas/:id/pagar — marcar paga e conciliar com movimento

POST /api/financas/extrato/import — upload CSV/CNAB para criar movimentos bancários

GET /api/financas/movimentos — listar movimentos importados

GET /api/financas/fluxo?inicio=YYYY-MM-DD&fim=YYYY-MM-DD — (implemente) resumo fluxo caixa por dia

⚛️ 4) Frontend React — páginas e componentes

Crie pasta frontend/src/pages/Financas/ com:

FinancasList.jsx — visão geral (contas)
import { useEffect, useState } from 'react';
import axios from 'axios';

export default function FinancasList() {
  const [contas, setContas] = useState([]);

  useEffect(() => {
    axios.get('/api/financas/contas')
      .then(r => setContas(r.data))
      .catch(err => console.error(err));
  }, []);

  return (
    <div className="p-4">
      <h2>Contas Financeiras</h2>
      <table className="w-full border">
        <thead>
          <tr><th>Tipo</th><th>Descrição</th><th>Valor</th><th>Vencimento</th><th>Status</th></tr>
        </thead>
        <tbody>
          {contas.map(c => (
            <tr key={c.id}>
              <td>{c.tipo}</td>
              <td>{c.descricao}</td>
              <td>R$ {c.valor}</td>
              <td>{c.data_vencimento}</td>
              <td>{c.status}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

ContaForm.jsx — criar conta pagar/receber
import { useState } from 'react';
import axios from 'axios';

export default function ContaForm({ onSaved }) {
  const [form, setForm] = useState({ tipo: 'receber', descricao:'', valor:'', data_vencimento:'' });

  const change = e => setForm({ ...form, [e.target.name]: e.target.value });

  const submit = async e => {
    e.preventDefault();
    await axios.post('/api/financas/contas', form);
    alert('Conta criada');
    onSaved && onSaved();
  };

  return (
    <form onSubmit={submit} className="p-3 bg-gray-100">
      <select name="tipo" value={form.tipo} onChange={change}>
        <option value="receber">Receber</option>
        <option value="pagar">Pagar</option>
      </select>
      <input name="descricao" placeholder="Descrição" value={form.descricao} onChange={change} />
      <input name="valor" type="number" step="0.01" name="valor" value={form.valor} onChange={change} />
      <input name="data_vencimento" type="date" value={form.data_vencimento} onChange={change} />
      <button type="submit">Salvar</button>
    </form>
  );
}

ExtratoUpload.jsx — importar extrato CSV simples
import { useState } from 'react';
import axios from 'axios';

export default function ExtratoUpload() {
  const [file, setFile] = useState(null);
  const [contaId, setContaId] = useState('');

  const submit = async e => {
    e.preventDefault();
    if (!file || !contaId) return alert('Escolha arquivo e conta bancária');
    const fd = new FormData();
    fd.append('file', file);
    fd.append('conta_bancaria_id', contaId);
    await axios.post('/api/financas/extrato/import', fd);
    alert('Importado');
  };

  return (
    <form onSubmit={submit}>
      <input type="file" accept=".csv" onChange={e => setFile(e.target.files[0])} />
      <input placeholder="Conta Bancária ID" value={contaId} onChange={e=>setContaId(e.target.value)} />
      <button type="submit">Importar</button>
    </form>
  );
}

🔗 5) Integrações e conciliação
Importar extratos

CNAB 240/400 é o padrão dos bancos; inicialmente comece com CSV exportado pelo internet banking (mais simples).

Para CNAB use bibliotecas específicas (Node: node-cnab ou cnabjs) para parse.

Conciliação automática

Estratégia: para cada MovimentoBancario, buscar ContaFinanceira com:

valor igual (ou aproximação),

data próxima,

descrição contendo referência do cliente/nota.

Se encontrar, atualizar contas_financeiras.status = 'pago' e relacionar movimentos_bancarios.conciliado_com_conta_id.

Open Banking / APIs bancárias

Open Banking permite puxar transações via API (mais seguro e automático) — cuidado com credenciais e segurança.

Inicialmente, comece com importação manual de CSV para validar processo.

🔒 6) Segurança e validações

Autenticação: JWT + permissões (financeiro role).

Validação: use Joi ou Zod nos endpoints (criarConta, marcarPago, importarExtratoCsv) para evitar dados inválidos.

Proteja uploads: limite tamanho, valide CSV.

Auditoria: registre user_id em alterações (quem marcou pago, quem importou extrato).

🧪 7) Testes rápidos

Unit: teste controllers com Jest + supertest (simule rotas).

Integração: testar importação CSV com amostras.

E2E: Cypress para fluxo "criar venda → gerar conta a receber → marcar pago via import de extrato".

Exemplo de teste super-básico (Jest + supertest):

import request from 'supertest';
import app from '../../src/main.js'; // seu express app

test('cria conta financeira', async () => {
  const res = await request(app)
    .post('/api/financas/contas')
    .send({ tipo:'receber', descricao:'Teste', valor:100.0, data_vencimento:'2025-01-01' });
  expect(res.statusCode).toBe(201);
  expect(res.body).toHaveProperty('id');
});

🧭 8) Deploy e operação

Banco: use um banco gerenciado; ative backups e replicas se disponível.

Armazenamento de extratos e XMLs: S3 (ou MinIO).

Cron jobs: agendamentos para reconciliar automaticamente (ex.: rodar job a cada 1h para tentar conciliar movimentos pendentes).

Logs & alertas: Sentry (erro), Prometheus/Grafana (métricas).

✅ 9) Checklist (rápido) para colocar Finanças em produção

 Tabelas criadas e migrations geradas.

 Endpoints testados (POST /contas, GET /contas, POST /contas/:id/pagar).

 Upload seguro de extratos (limite e validação).

 Regras de conciliação definidas (match por valor/data/descrição).

 Roles e permissões implementadas.

 Backups e logs configurados.

 Testes automatizados (unit + integração).
