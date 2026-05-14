# Security Architecture — Snake Game Leaderboard

Este documento detalha as medidas de segurança implementadas para garantir a integridade do sistema de classificação (leaderboard), protegendo-o contra manipulações de pontuação, ataques de injeção e abusos de automação.

## 1. Visão Geral da Estratégia de Defesa
A arquitetura adota o princípio de **Defesa em Profundidade (Defense in Depth)**, onde falhas numa camada de controlo são mitigadas por camadas subsequentes. O sistema transita de um modelo de "confiança no cliente" para um modelo de **Validação Defensiva Estrita**.

### Fluxo de Validação de Dados
1.  **Ingress:** Sanitização e Normalização de Input (Frontend).
2.  **Transport:** Validação de Schema e Tipagem (API).
3.  **Processing:** Heurísticas Anti-Cheat e Rate Limiting (Backend).
4.  **Egress:** Parsing Defensivo e Renderização Segura (Frontend).



---

## 2. Controles de Segurança de Input (Nome do Jogador)

### Sanitização e Normalização
Toda entrada de texto passa por um pipeline de limpeza antes de qualquer processamento:
* **Normalização Unicode (NFKC):** Converte caracteres visualmente equivalentes (ex: `ＡＤＭＩＮ` para `ADMIN`) para evitar *spoofing* e ataques de homógrafos.
* **Remoção de Caracteres Invisíveis/Bidi:** Elimina caracteres de controlo bidirecional e *zero-width spaces* que poderiam ser usados para burlar identidades ou manipular a ordem visual do texto.
* **Allowlist Estrita:** Aplicação de Regex `/[^A-Za-zÀ-ÖØ-öø-ÿ0-9 _-]/g`. Apenas caracteres alfanuméricos, espaços, underscores e hífens são permitidos.

### Mitigações
* **Stored XSS & HTML Injection:** O payload perde capacidade de execução ao ser sanitizado e limpo de tags.
* **Visual Impersonation:** Impede que atacantes se passem por outros utilizadores usando caracteres especiais idênticos.

---

## 3. Integridade do ID e Estado

### Validação de Identificador
O ID do jogador utiliza um charset restrito `[A-HJ-NP-Z2-9]{6}`.
* **Exclusão de Caracteres Ambíguos:** Removidos `O, 0, I, 1` para prevenir confusão visual e ataques de engenharia social.
* **Validação de Formato:** IDs que não seguem o tamanho fixo ou charset são rejeitados imediatamente.

---

## 4. Mecanismos Anti-Cheat e Validação de Pontuação

### Regras de Negócio e Heurísticas
Diferente de sistemas legados, o backend não aceita pontuações de forma passiva:
* **Validação de Step:** O score deve ser múltiplo de 10 (regra do jogo). Valores como `777` são sumariamente rejeitados.
* **Heurística de Tempo (Temporal Analysis):** O backend valida a relação `Score vs Duração da Partida`. Se um jogador atinge 500 pontos em 100ms, o sistema identifica como impossibilidade física e descarta o registro.
* **Constraints de Range:** Definição estrita de valores mínimos (0) e máximos (5000) plausíveis.

### Persistência Autoritativa
* **Server-Side Cooldown:** O controlo de frequência de envio (cooldown) é validado no servidor. O bypass via limpeza de `localStorage` é ineficaz.
* **Idempotência e Melhor Score:** O sistema mantém apenas a melhor pontuação por ID, prevenindo o *flooding* da base de dados com entradas redundantes.

---

## 5. Segurança de Infraestrutura e API

### Rate Limiting
Implementação de limites de requisições baseados em:
* **IP Source:** Previne ataques de negação de serviço (DoS) e brute force.
* **Player ID:** Impede que um único utilizador automatize o envio de milhares de scores através de scripts.

### Parsing Defensivo (Frontend)
O frontend trata a resposta da API como **não confiável**.
* **Schema Validation:** O método `parseScoresApiResponse()` valida se cada objeto da lista contém os tipos e formatos esperados antes de atualizar o estado do React.
* **Type Safety em Runtime:** O uso de TypeScript é reforçado por validações explícitas, prevenindo *Type Confusion* ou quebras de estado por JSON malformado.

---

## 6. Matriz de Mitigação de Riscos

| Ameaça | Técnica de Mitigação | Status |
| :--- | :--- | :--- |
| **Stored XSS** | Sanitização de Input + React Escaping | Protegido |
| **Score Forgery** | Validação de Step + Heurística Temporal | Mitigado |
| **Unicode Spoofing** | Normalização NFKC | Protegido |
| **Bidi Attacks** | Remoção de caracteres de controlo | Protegido |
| **API Spam** | Rate Limiting + Cooldown Server-side | Mitigado |
| **State Poisoning** | Parsing Defensivo de API | Protegido |

---

## 7. Limitações Conhecidas
Embora a superfície de ataque tenha sido drasticamente reduzida, este sistema opera sob um modelo de execução client-side. Um atacante sofisticado que realize engenharia reversa do binário/script e emule perfeitamente o comportamento humano (respeitando tempos de resposta e protocolos) ainda representa um risco residual. O foco desta arquitetura é eliminar abusos triviais e automatizações em escala.
