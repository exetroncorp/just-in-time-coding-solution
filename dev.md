# Dépannage : AI gateway et SSH over HTTPS (Kubernetes, JupyterHub)

Pour l'image `nanto2vison/devx-studio:xpra61` (= `latest`) et `:xpra61-bench`.
Les commandes se lancent dans un **terminal du bureau**, ou depuis ton poste avec
`kubectl -n <namespace> exec -it jupyter-<user> -- bash -l`.

---

## 1. L'AI gateway (agentgateway) ne démarre pas

### Comment il démarre

- supervisord lance le programme `ai-gateway` (`/opt/devdesk/supervisor/core.conf`) :
  `command=/opt/devdesk/libexec/start-ai-gateway`
- ce script fait : `exec agentgateway -f /tmp/devdesk/etc/agentgateway/config.yaml`
- **au démarrage du pod**, l'entrypoint (`devdesk-config apply`) écrit tout `/tmp/devdesk/etc/`, dont ce
  `config.yaml`. Il l'écrit **vide** quand `AI_GATEWAY_URL` n'est pas réglé, et ne met
  `autostart` à vrai que si `AI_GATEWAY_URL` est réglé (et `AI_GATEWAY_LOCAL` pas à `false`).
  Que `/tmp` soit un PVC ne change rien : tout y est régénéré à chaque démarrage.
- sa sortie va dans **le log du pod** (« Show full log » de JupyterHub, ou `kubectl logs`).
- ne lance jamais `agentgateway` seul : sans `-f`, il répond « configuration is required when running in
  Kubernetes ». C'est normal.

### Les commandes, dans l'ordre

```bash
# 1. Est-il configuré ?  (URL vide ou config.yaml de 0 octet = passerelle éteinte)
echo "URL=$AI_GATEWAY_URL LOCAL=${AI_GATEWAY_LOCAL:-true} MODEL=$AI_GATEWAY_MODEL AUTOSTART=$DEVDESK_AI_GATEWAY_AUTOSTART"
wc -c /tmp/devdesk/etc/agentgateway/config.yaml
devdesk-settings get AI_GATEWAY_URL

# 2. Son état
supervisorctl -c /opt/devdesk/supervisor/supervisord.conf status ai-gateway

# 3. Le lancer à la main, comme au démarrage, au premier plan (Ctrl+C pour arrêter)
supervisorctl -c /opt/devdesk/supervisor/supervisord.conf stop ai-gateway
/opt/devdesk/libexec/start-ai-gateway
# ... puis le rendre à supervisord :
supervisorctl -c /opt/devdesk/supervisor/supervisord.conf start ai-gateway

# 4. Répond-il ?
curl -s http://127.0.0.1:15021/healthz/ready; echo
curl -s --noproxy '*' http://127.0.0.1:4000/v1/models | head -c 300; echo
devdesk-doctor --only gateway

# 5. Ton endpoint d'entreprise, sans passer par agentgateway (le jeton ne s'affiche pas)
printf 'Authorization: Bearer %s\n' "$(cat /etc/devdesk/secrets/ai-gateway-token)" > /tmp/h && chmod 600 /tmp/h
curl -sS -o /dev/null -w 'via le proxy : %{http_code}\n' -H @/tmp/h "$AI_GATEWAY_URL/models"
curl -sS -o /dev/null -w 'en direct    : %{http_code}\n' --noproxy '*' -H @/tmp/h "$AI_GATEWAY_URL/models"
rm -f /tmp/h
```

### Ce que tu vois, ce que ça veut dire

| Tu vois | Cause | À faire |
| --- | --- | --- |
| `URL=` vide, `config.yaml` à 0 octet | aucune passerelle réglée | Settings > AI gateway, ou `'AI_GATEWAY_URL'` + `'AI_GATEWAY_MODEL'` dans `environment` du profil KubeSpawner, jeton dans le Secret monté en `/etc/devdesk/secrets/ai-gateway-token` |
| `STOPPED  Not started` alors que l'URL est réglée | l'URL a été réglée après le démarrage du pod | `supervisorctl … start ai-gateway`, ou applique le réglage dans Settings (il le démarre) |
| `BACKOFF` / `FATAL` | il plante au démarrage | étape 3 : l'erreur s'affiche ; sinon les lignes `ai-gateway` du log du pod |
| `ready` OK, `/v1/models` en erreur | l'endpoint d'entreprise refuse ou est injoignable | étape 5 |
| étape 5 : `401` / `403` | jeton absent ou faux | vérifie le Secret `ai-gateway-token` |
| étape 5 : `000` + `SSL certificate problem` (ajoute `-v`) | CA d'entreprise inconnue | Settings > Certificates : importe la **racine** (et l'intermédiaire) ; voir § 3 |
| étape 5 : `000` « via le proxy », `200` « en direct » | l'endpoint est interne et ne doit pas passer par le proxy | Settings : ajoute son nom dans `NO_PROXY_EXTRA` |
| étape 5 : `407` | le proxy d'entreprise demande une authentification | Settings : proxy amont (`UPSTREAM_PROXY_URL`) et ses identifiants |

---

## 2. SSH over HTTPS : 403 derrière JupyterHub

### Comment c'est routé

Le port web du bureau (14500) passe par un petit gateway. **Seul le chemin exact `/devx/ssh`**
(WebSocket + jeton du tunnel) va vers `sshd`. Tout le reste va vers Xpra.

### La cause probable (vérifiée sur xpra61)

Derrière JupyterHub, la requête arrive avec son préfixe : `/user/userid/devx/ssh` ou
`/user/userid/https/14500/devx/ssh`. Le gateway ne la reconnaît pas et l'envoie à Xpra, qui répond :

```
HTTP/1.0 403 failed to handle websocket: client does not support 'binary' protocol
Server: Xpra-WebSocket-Server Python/3.13
```

Reproduit sur un bureau xpra61, SSH activé :

| Chemin demandé | Réponse |
| --- | --- |
| `/devx/ssh` | `401` « a tunnel token is needed » : c'est le gateway SSH, c'est normal sans jeton |
| `/user/userid/devx/ssh` | **`403`** Xpra « binary protocol » |
| `/user/userid/https/14500/devx/ssh` | **`403`** Xpra « binary protocol » |

Correction côté image : accepter tout chemin qui **finit** par `/devx/ssh`. En attendant, demande que ton
proxy retire le préfixe pour ce chemin (`/user/<nom>/devx/ssh` vers `/devx/ssh`).

### Étape 0 : SSH est-il activé dans le bureau ?

```bash
devdesk-ssh status
devdesk-settings get SSH_ENABLED
supervisorctl -c /opt/devdesk/supervisor/supervisord.conf status ssh-gateway
```

`SSH_ENABLED` doit valoir `true` : dans Settings, ou `'SSH_ENABLED': 'true'` dans `environment` du profil.
`SSH_PUBLIC_URL` = l'adresse vue depuis ton poste, par exemple `https://myenv.corp/user/userid`.

### Étape 1 : dans le pod, le tunnel répond-il ?

```bash
W='-H Connection:Upgrade -H Upgrade:websocket -H Sec-WebSocket-Version:13 -H Sec-WebSocket-Key:dGhlIHNhbXBsZSBub25jZQ=='
curl -s -i -m 5 --noproxy '*' $W http://127.0.0.1:14500/devx/ssh | head -1                 # attendu : 401
curl -s -i -m 5 --noproxy '*' $W http://127.0.0.1:14500/user/userid/devx/ssh | head -1     # 403 Xpra = le préfixe
```

### Étape 2 : depuis ton poste, qui répond ?

Mets le jeton dans un fichier, sans l'afficher : dans le bureau, `devdesk-ssh token` le copie dans le
presse-papiers. Colle-le dans `~/devx.token` sur ton poste.

**Git Bash / Linux / macOS :**

```bash
for U in https://myenv.corp/user/userid/devx/ssh https://myenv.corp/user/userid/https/14500/devx/ssh; do
  echo "== $U"
  curl -s -i -m 10 --http1.1 -H "Connection: Upgrade" -H "Upgrade: websocket" -H "Sec-WebSocket-Version: 13" \
       -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" -H "Authorization: Bearer $(cat ~/devx.token)" "$U" \
    | tr -d '\r' | grep -i -E '^HTTP/|^server:|^via:|^x-jupyterhub|^location:|^www-authenticate:|tunnel|websocket|binary' | head -8
done
```

**PowerShell (Windows) :**

```powershell
$t = (Get-Content "$HOME\devx.token" -TotalCount 1).Trim()
foreach ($u in 'https://myenv.corp/user/userid/devx/ssh','https://myenv.corp/user/userid/https/14500/devx/ssh') {
  "== $u"
  curl.exe -s -i -m 10 --http1.1 -H "Connection: Upgrade" -H "Upgrade: websocket" -H "Sec-WebSocket-Version: 13" `
    -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" -H "Authorization: Bearer $t" $u | Select-Object -First 12
}
```

### Étape 3 : lire la réponse

| Réponse | Qui répond | À faire |
| --- | --- | --- |
| `101 Switching Protocols` | le tunnel : tout va bien | le souci est dans la configuration de VS Code (étape 4) |
| `403` + `Server: Xpra-WebSocket-Server` + « binary protocol » | **Xpra** : le préfixe n'est pas retiré | correction de l'image, ou réécriture du chemin par le proxy (voir plus haut) |
| `403 Forbidden` « the SSH tunnel is not for web pages » | le gateway : un proxy ajoute un en-tête `Origin` | que le proxy n'ajoute pas `Origin` sur ce chemin |
| `401` « a tunnel token is needed » | le gateway : jeton absent ou faux | refais l'étape 2 avec le bon jeton ; si un proxy garde `Authorization` pour lui, mets `"tokenHeader": "X-Devx-Tunnel-Token"` (étape 4) |
| `404` « SSH over HTTPS is off » | le gateway : SSH désactivé | étape 0 |
| `426` « this is a WebSocket endpoint » | le gateway : un proxy a supprimé `Upgrade` | le proxy doit laisser passer les WebSocket |
| page HTML JupyterHub (`X-JupyterHub-Version`), ou `302` vers une page de connexion | **JupyterHub ou le SSO** : la requête n'atteint pas le pod | les admins doivent laisser passer les clients non-navigateurs sur `/user/*/devx/ssh` |
| page HTML d'un pare-feu ou d'un WAF d'entreprise | l'ingress d'entreprise | faire autoriser les WebSocket vers ce chemin |
| `407 Proxy Authentication Required` | le proxy de ton poste | l'extension ne sait pas s'authentifier auprès d'un proxy : il faut un proxy sans authentification ou une exception |

### Étape 4 : la configuration de l'extension VS Code

Fichier `~/.devx/ssh/<NOM>.json` sur ton poste (le jeton est dans `<NOM>.token`, à côté) :

```json
{
  "name": "myenv",
  "url": "https://myenv.corp/user/userid",
  "path": "/devx/ssh",
  "user": "dev",
  "folder": "/home/dev/workspace",
  "tokenHeader": "X-Devx-Tunnel-Token"
}
```

- `url` peut porter le préfixe : l'extension appelle `url` + `path`, soit `/user/userid/devx/ssh`.
- `tokenHeader` n'est utile que si un proxy garde l'en-tête `Authorization` pour lui. Sinon, retire la ligne.

Puis, dans un terminal du poste :

```bash
ssh -v devx-myenv 2>&1 | grep -i -E 'devx-ssh-proxy|HTTP [0-9]{3}|refused|denied' | head
```

Le message d'erreur de l'extension donne le chemin appelé et le code HTTP. Compare-le au tableau de l'étape 3.
