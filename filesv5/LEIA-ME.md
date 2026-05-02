# ZerouGAME — Fixes Aplicados

## ✅ O que foi corrigido

### Fix ① — Nome do utilizador não aparecia
**Causa:** O registo fazia `showScreen('login')` manualmente, mas o
`onAuthStateChanged` ainda não tinha o nome guardado no Firebase Auth.

**Solução aplicada em `app.js`:**
```js
// Após criar a conta, guardar o displayName no Firebase Auth
await cred.user.updateProfile({ displayName: nome });

// E no Realtime Database guardar explicitamente
const userData = { nome, email, ... };
await firebase.set(ref(db, `users/${uid}`), userData);

// NÃO redirecionar para login manualmente —
// o onAuthStateChanged detecta o login e trata tudo automaticamente
```

---

### Fix ② — Saques desapareciam ao sair e voltar
**Causa:** Os saques só eram guardados em `/saques/{id}` (nó separado).
A query `orderByChild('userId').equalTo(uid)` precisava de um **índice
Firebase** que não existia → a query falhava silenciosamente e o
histórico aparecia vazio.

**Solução aplicada:**
- Os saques agora são guardados em **dois sítios**:
  1. `/users/{uid}/saques/{saqueId}` → pertence ao perfil, carrega sempre com o listener
  2. `/saques/{saqueId}` → para o admin ver todos
- O listener `onValue` aponta para `/users/{uid}` (sem query) → sem erro de índice

```js
// Gravar saque em dois sítios
await firebase.update(ref(db, `users/${uid}/saques/${saqueId}`), saqueData);
await firebase.set(ref(db, `saques/${saqueId}`), saqueData);
```

---

### Fix ③ — Status "pago" em tempo real
**Solução:** O admin usa `update()` atómico que escreve nos dois sítios
em simultâneo. O listener do utilizador detecta a mudança e mostra
notificação + som.

```js
// Admin — update atómico nos dois nós
const updates = {};
updates[`saques/${saqueId}/status`]                 = 'pago';
updates[`users/${userId}/saques/${saqueId}/status`] = 'pago';
await update(ref(db), updates); // um único write
```

---

## 📁 Ficheiros fornecidos

| Ficheiro | O que fazer |
|---|---|
| `app.js` | Substituir o `app.js` actual na pasta do projecto |
| `firebase-rules.json` | Copiar para **Firebase Console → Realtime Database → Rules** |
| `admin.html` | Abrir num browser para gerir saques |

---

## 🔧 Como aplicar no Firebase Console

### 1. Regras do Realtime Database
1. Vai a **Firebase Console → Realtime Database → Rules**
2. Substitui TODO o conteúdo pelo ficheiro `firebase-rules.json`
3. Clica **Publish**

### 2. Testar
1. Cria uma conta nova na app
2. Verifica que o nome aparece correctamente na home
3. Faz um saque (mínimo 50 Kz)
4. Abre `admin.html` e clica "✅ Pago"
5. Na app, deves ver a notificação "✅ Saque de X Kz foi PAGO!" em tempo real
6. Sai da app e volta → o saque ainda aparece no histórico

---

## ⚠️ Regras para perguntas no Firebase

As tuas perguntas devem ter esta estrutura:
```json
{
  "perguntas": {
    "p001": {
      "texto": "Qual é a capital de Angola?",
      "categoria": "Geografia",
      "opcoes": ["Luanda", "Benguela", "Huambo", "Lubango"],
      "respostaIndex": 0
    }
  }
}
```
**Campos aceites:** `texto` ou `pergunta` · `respostaIndex` ou `resposta` (índice 0-3)
