# Morning Call — Contexto do Projeto

## Objetivo
Automatizar a publicação diária de um **Instagram Story** com o resumo do Morning Call financeiro.

---

## Dois fluxos distintos

### Fluxo 1 — Dias Úteis (em desenvolvimento futuro)
Coleta conteúdo de 3 fontes, sintetiza via Claude e gera imagem para postar.
- **XP Investimentos** → RSS feed
- **BTG Pactual** → YouTube transcript via Supadata API
- **UBS CIO Daily** → scraping do site UBS

### Fluxo 2 — Fins de semana e Feriados ← *foco atual*
Posta uma imagem simples com a data do dia e uma dica engraçada gerada por Claude.

---

## Arquitetura do Fluxo 2 (MC-sem_info)

```
Schedule Trigger (09:00)
  → Feriados API (brasilapi.com.br)
  → Check Feriado (Code node)
  → IF Dia Útil?
      └── FALSE → [fluxo abaixo]
```

```
Formata Data PT-BR
  → Claude API (gera dica engraçada contextualizada ao dia)
  → Extrai Dica
  → Gera HTML (1080×1920 com imagem de fundo em base64)
  → htmlcsstoimage.com (converte HTML → PNG)
  → Extrai URL PNG
  → [futuro: Instagram Graph API]
```

---

## Imagem gerada

- **Tamanho:** 1080×1920px (Instagram Story 9:16)
- **Fundo:** PNG fixo (`background.png`) embutido como base64 no HTML
- **Textos dinâmicos:**
  - Data: `20 de junho de 2026 às 09:00` — SF Pro Text 30px, `#999999`, centralizado
  - Frase fixa: `Hoje não tem dados de mercado.` — SF Pro Text 42px, `#ffffff`
  - Dica (itálico): gerada pelo Claude — SF Pro Text 40px, `rgba(255,255,255,0.55)`
- **Tipografia:** Apple HIG (SF Pro Text, letter-spacing negativo, line-height 1.3)

### Posições no canvas 1080×1920
| Elemento | top | left |
|---|---|---|
| Data | 282px | centralizado |
| Frase principal | 482px | 73px |
| Dica | 620px | 73px |

---

## Infraestrutura

- **n8n:** self-hosted via Docker no servidor `kirk`
- **Container:** `n8nio/n8n:latest`
- **Dados:** volume em `/opt/n8n/data`
- **URL:** `https://n8n.brunoalmeida.adv.br`

---

## Tarefa pendente — configurar variável de ambiente

A imagem de fundo precisa estar disponível no n8n como variável de ambiente `MC_BACKGROUND_BASE64`.

### Passo 1 — No Windows (já feito)
```powershell
# Gerar o base64 e salvar em arquivo
[Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\Users\bruno.almeida\Desktop\mc\background.png")) | Out-File -NoNewline -FilePath "C:\Users\bruno.almeida\Desktop\mc\bg.b64"

# Enviar para o servidor
scp C:\Users\bruno.almeida\Desktop\mc\bg.b64 brunosalme@kirk:/opt/n8n/mc_background.b64
```

### Passo 2 — No kirk (pendente)
Recriar o container n8n adicionando a variável `MC_BACKGROUND_BASE64`:

```bash
docker stop n8n && docker rm n8n && docker run -d \
  --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -v /opt/n8n/data:/home/node/.n8n \
  -e N8N_ENCRYPTION_KEY=<sua_chave_atual> \   # copie do docker inspect n8n
  -e GENERIC_TIMEZONE=America/Sao_Paulo \
  -e TZ=America/Sao_Paulo \
  -e N8N_HOST=0.0.0.0 \
  -e N8N_PORT=5678 \
  -e N8N_SECURE_COOKIE=true \
  -e N8N_PROTOCOL=https \
  -e "WEBHOOK_URL=https://n8n.brunoalmeida.adv.br" \
  -e "MC_BACKGROUND_BASE64=$(cat /opt/n8n/mc_background.b64)" \
  n8nio/n8n:latest
```

---

## Próximos passos após variável configurada

1. **Criar conta** em [htmlcsstoimage.com](https://htmlcsstoimage.com) (free tier: 50 imagens/mês)
2. **Criar credencial** no n8n: `HTTP Basic Auth` com User ID + API Key do HCTI
3. **Importar nós** do arquivo `morning-call-sem-info.json` (repositório `brunosalme/brunosalme`, branch `claude/quirky-brown-xsddbv`)
4. **Conectar** o ramo FALSE do `IF Dia Útil?` ao nó `Formata Data PT-BR`
5. **Testar** execução manual
6. **Configurar Instagram Graph API** para postagem automática do Story

---

## Arquivos no repositório

| Arquivo | Descrição |
|---|---|
| `morning-call-sem-info.json` | Nós n8n do fluxo Fluxo 2 prontos para importar |
| `mc-sem-info-final.html` | Preview HTML do layout (para ajustes visuais) |

**Repositório:** `brunosalme/brunosalme` — branch `claude/quirky-brown-xsddbv`

---

## Variáveis necessárias no n8n

| Variável | Valor |
|---|---|
| `MC_BACKGROUND_BASE64` | base64 do `background.png` (via env var no Docker) |
| `ANTHROPIC_API_KEY` | Chave da API Anthropic |
| `HCTI_USER_ID` | User ID do htmlcsstoimage.com |
| `HCTI_API_KEY` | API Key do htmlcsstoimage.com |
