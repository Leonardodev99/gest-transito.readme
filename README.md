# GTL — Gestão de Trânsito em Luanda
https://github.com/Leonardodev99/gest-transito.readme/blob/main/LICENSE
## Documentação Técnica
**Versão do documento:** 1.0
**Última atualização:** Julho de 2026

## 1. Visão Geral

O GTL é um sistema integrado de gestão de trânsito composto por três aplicações que partilham um único backend:

| Componente | Descrição | Público-alvo |
|---|---|---|
| **Backend (API)** | API REST + WebSocket em Node.js/Express | Serve as outras duas aplicações |
| **Central (Web)** | Painel administrativo em React | Administradores |
| **App Mobile** | Aplicação em React Native (Expo) | Agentes de trânsito em campo |

O sistema cobre: gestão de agentes, condutores e veículos; registo e consulta de ocorrências, multas, apreensões e pagamentos; relatórios estatísticos; localização de agentes em tempo real; e consulta de documentos (BI, carta de condução, veículo) em campo.
## 2. Arquitetura do Sistema

```
┌─────────────────────┐         ┌──────────────────────┐
│   Central (Web)      │◄───────►│                       │
│   React + Vite        │  REST   │                       │
│   porta 5173           │  +      │                       │
│                       │  WS     │      Backend API      │
└─────────────────────┘         │   Node.js + Express    │
                                  │   porta 3001           │
┌─────────────────────┐         │                       │
│   App Mobile          │◄───────►│   Socket.io (WS)       │
│   React Native (Expo) │  REST   │                       │
│                       │         │                       │
└─────────────────────┘         └───────────┬───────────┘
                                              │
                                              ▼
                                  ┌───────────────────────┐
                                  │   Base de Dados         │
                                  │   (via Sequelize ORM)  │
                                  └───────────────────────┘
```
### Por que essa arquitetura

- **Um único backend para dois clientes**: evita duplicação de regras de negócio (validações, permissões, cálculos) entre a central e o app mobile.
- **Autorização por papel (`role`)**: o mesmo endpoint pode ter comportamento diferente conforme o token pertence a um `admin` ou a um `agente` — controlado por middleware, não por servidores separados.
- **WebSocket só para um caso de uso**: localização de agentes em tempo real. Todo o resto do sistema é REST tradicional (polling manual/refresh), porque não há necessidade de latência mínima nesses fluxos.

---
## 3. Stack Tecnológica

### 3.1 Backend

| Camada | Tecnologia |
|---|---|
| Runtime | Node.js |
| Framework HTTP | Express |
| ORM | Sequelize (migrations + models) |
| Autenticação | JWT (`jsonwebtoken`) |
| Hash de senha | bcryptjs |
| Upload de arquivos | Multer (whitelist de mimetypes: PNG, JPG, JPEG, PDF) |
| Tempo real | Socket.io |
| E-mail | Nodemailer (via `transporter` configurado em `config/mail.js`) |
| CORS | pacote `cors`, com whitelist de origem |

### 3.2 Central (Web)

| Camada | Tecnologia |
|---|---|
| Framework | React |
| Bundler | Vite |
| HTTP client | Axios |
| Roteamento | React Router |
| Mapa | React Leaflet + OpenStreetMap |
| Tempo real | Socket.io-client |
| Gráficos | Recharts |
| Exportação | jsPDF + jspdf-autotable (PDF), SheetJS/xlsx (Excel), file-saver |
| Renderização de gráfico p/ PDF | html2canvas |

### 3.3 App Mobile

| Camada | Tecnologia |
|---|---|
| Framework | React Native |
| Plataforma | Expo (Expo Go / development build) |
| Roteamento | Expo Router (baseado em arquivos) |
| HTTP client | Axios |
| Armazenamento local | AsyncStorage |
| Localização | expo-location |
| Geração de PDF | expo-print |
| Compartilhamento de arquivo | expo-sharing |
| Sistema de arquivos | expo-file-system (API legada, `/legacy`) |
| Gráficos | react-native-chart-kit |
| Ícones | @expo/vector-icons (MaterialIcons) |
| Conversão binário↔base64 | buffer |
---

## 4. Modelo de Dados

### 4.1 Entidades principais

```
Admin ──1:N──► Agente ──1:N──► Ocorrencia ──1:N──► Multa ──1:N──► Pagamento
                                    │
                                    ├──N:1──► Condutor
                                    └──N:1──► Veiculo ──N:1──► Condutor

Agente ──1:N──► ConsultaLog
```
## 5. Autenticação e Autorização

### 5.1 Login

`POST /tokens` recebe `{ email, senha }`, procura primeiro em `Admin`, depois em `Agente`, valida a senha com `bcrypt.compare`, e devolve um JWT assinado (`TOKEN_SECRET`, expiração `TOKEN_EXPIRATION`) contendo `{ id, type }`.

### 5.2 Middleware de autenticação

Todas as rotas protegidas passam por `authMiddleware`, que:
1. Extrai o token do header `Authorization: Bearer <token>`
2. Verifica a assinatura
3. Injeta `req.user = { id, type }` na requisição

### 5.3 Autorização por papel

Cada rota decide, individualmente, quem pode acessá-la:

| Recurso | Admin | Agente |
|---|---|---|
| Gerir Agentes/Condutores/Veículos (CRUD) | ✅ | ❌ (exceto perfil próprio) |
| Registar Ocorrência/Multa/Apreensão | ❌ | ✅ |
| Listar tudo (index) | ✅ | ❌ |
| Consultar/buscar (BI, carta, veículo, multas, ocorrências) | — | ✅ |
| Atualizar status de Pagamento | ✅ | ❌ |
| Ver relatórios | ✅ | — |
| Enviar/consultar localização | — (só consulta) | ✅ (só envia) |
## 6. Segurança

### 6.1 Medidas implementadas

- **Hash de senha**: bcrypt (custo 8), nunca armazenada em texto plano.
- **Campos sensíveis excluídos por padrão**: `defaultScope` do Sequelize remove `senha_hash`, `password_reset_token` e `password_reset_code` de qualquer consulta, a menos que explicitamente solicitado via `Model.scope(null)`.
- **Confirmação de senha atual**: alterar a própria senha (admin ou agente) exige informar a senha atual, validada via `bcrypt.compare` antes de aceitar a nova.
- **Tokens/códigos de redefinição de senha com hash**: tanto o token longo (link, admin) quanto o código curto (6 dígitos, agente) são armazenados como hash SHA-256, nunca em texto plano — comparação sempre feita hash-a-hash.
- **Expiração de redefinição de senha**: 15 minutos, verificado no momento do reset.
- **Validação de upload**: `multer` restringe uploads a `image/png`, `image/jpg`, `image/jpeg` e `application/pdf`.
- **CORS com whitelist**: apenas origens explicitamente permitidas (central web) podem fazer requisições cross-origin; requisições sem header `Origin` (apps mobile nativos) passam livremente.
- **WebSocket autenticado**: conexões Socket.io exigem o mesmo JWT das rotas REST, verificado no handshake antes de aceitar a conexão.
- **Proteção contra SQL injection**: todo acesso a dados passa pelo Sequelize ORM (queries parametrizadas), sem SQL bruto concatenado com input do usuário.
## 7. Comunicação em Tempo Real (WebSocket)

**Objetivo:** mostrar a posição dos agentes no mapa da central instantaneamente, sem recarregar a página.

**Fluxo:**
1. Central conecta ao Socket.io com o JWT no handshake (`auth: { token }`).
2. Se `type === 'admin'`, o socket entra na sala `'admins'`.
3. App mobile do agente chama `PUT /agentes/localizacao` (REST, não WebSocket) periodicamente — evita exigir um cliente Socket.io no app mobile.
4. O backend salva a posição no banco **e** emite `agente:localizacao-atualizada` para a sala `'admins'`.
5. A central escuta esse evento e atualiza o marcador no mapa em tempo real.
## 15. Estrutura de Pastas (referência)

```
backend/
├── app.js
├── server.js
├── src/
│   ├── database/          # conexão + registro de models
│   ├── models/             # Admin, Agente, Condutor, Veiculo,
│   │                        # Ocorrencia, Multa, Pagamento, ConsultaLog
│   ├── controllers/
│   ├── routes/
│   ├── middlewares/        # auth.js, upload.js
│   ├── config/              # mail.js, multerConfig.js
│   ├── utils/                # consultaLogger.js
│   └── socket.js            # inicialização do Socket.io
└── migrations/

central-web/
├── src/
│   ├── pages/                # Dashboard, AgentManagement, Map, Relatorios...
│   ├── services/             # api.js, socketClient.js
│   └── styles/

app-mobile/
├── app/                      # rotas (Expo Router)
│   ├── login.jsx
│   ├── agentDashboard.jsx
│   ├── novaOcorrencia.jsx
│   ├── consultarBI.jsx / consultarCarta.jsx / consultarVeiculo.jsx / consultarMultas.jsx / consultarOcorrencias.jsx
│   ├── esqueciSenha.jsx / codeSent.jsx / passwordReset.jsx / newPassword.jsx
│   └── agentProfile.jsx
└── src/
    ├── services/              # api.js
    ├── hooks/                 # useLocationTracking.js
    └── utils/                  # documentUtils.js
```

---
