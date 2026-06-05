# Serviços de PLN no Google Cloud

Levantamento dos serviços de Processamento de Linguagem Natural (PLN) disponíveis no Google Cloud Platform, com descrição de cada serviço e exemplos de aplicação via requisição HTTP, contextualizados ao projeto desenvolvido para a **ZAMP** (holding de food service que opera Burger King, Subway, Popeyes e Starbucks no Brasil).

O Google Cloud foi escolhido como nuvem de referência deste levantamento porque **o próprio parceiro ZAMP já utiliza a infraestrutura Google**: os dados de clientes e workloads de ML do projeto estão hospedados no GCP, o que torna a adoção dos serviços de PLN nativos da plataforma uma extensão natural da arquitetura existente. Além disso, o ecossistema de PLN do Google Cloud é um dos mais completos do mercado, abrangendo desde análise estruturada de texto com modelos pré-treinados (Cloud Natural Language API) até agentes conversacionais (Dialogflow CX), síntese e reconhecimento de fala (Text-to-Speech e Speech-to-Text), tradução automática neural (Translation API) e modelos de linguagem de grande porte via Vertex AI (família Gemini).

> Todos os exemplos utilizam requisições HTTP diretas, sem bibliotecas específicas do fabricante.

---

## 1. Cloud Natural Language API

**Endpoint base:** `https://language.googleapis.com/v1/`

O Cloud Natural Language API expõe modelos pré-treinados do Google para análise de texto em PT-BR e mais de 10 outros idiomas. Oferece cinco capacidades principais acessíveis via REST:

| Recurso | Descrição |
|---|---|
| Análise de Sentimento | Retorna score (-1 a +1) e magnitude para o documento e para cada sentença |
| Reconhecimento de Entidades | Identifica e classifica entidades (pessoa, local, organização, produto etc.) com links para o Knowledge Graph |
| Análise Sintática | Tokenização, part-of-speech tagging e árvore de dependência |
| Sentimento por Entidade | Combina reconhecimento de entidades com análise de sentimento por entidade |
| Classificação de Conteúdo | Classifica o documento em mais de 700 categorias hierárquicas predefinidas |

---

### 1.1 Análise de Sentimento

Retorna um `score` de -1,0 (muito negativo) a +1,0 (muito positivo) e uma `magnitude` que representa a intensidade emocional total do texto, independente da polaridade.

**Aplicação no contexto ZAMP:** classificar automaticamente avaliações do Google Reviews e do Reclame Aqui das lojas Burger King, Subway, Popeyes e Starbucks, disparando alertas para comentários com score abaixo de -0,5.

```bash
curl -X POST \
  "https://language.googleapis.com/v1/documents:analyzeSentiment?key=SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "document": {
      "type": "PLAIN_TEXT",
      "language": "pt",
      "content": "Absurdo! Esperei 40 minutos e o lanche chegou frio e sem o molho. Nunca mais volto."
    },
    "encodingType": "UTF8"
  }'
```

Resposta:
```json
{
  "documentSentiment": {
    "score": -0.9,
    "magnitude": 2.1
  },
  "sentences": [
    {
      "text": { "content": "Absurdo! Esperei 40 minutos e o lanche chegou frio e sem o molho." },
      "sentiment": { "score": -0.9, "magnitude": 1.8 }
    },
    {
      "text": { "content": "Nunca mais volto." },
      "sentiment": { "score": -0.8, "magnitude": 0.8 }
    }
  ]
}
```

---

### 1.2 Reconhecimento de Entidades

Identifica e classifica entidades no texto (pessoas, organizações, locais, produtos) e retorna metadados como salience (relevância) e, quando disponível, links para o Google Knowledge Graph.

**Aplicação no contexto ZAMP:** desambiguar comentários multi-marca. A ZAMP opera quatro franquias que frequentemente aparecem juntas em praças de alimentação. Um comentário pode mencionar BK e Subway simultaneamente, dificultando a atribuição de sentimento por marca. A API identificaria cada organização como entidade distinta, permitindo separar a qual franquia cada trecho pertence.

```bash
curl -X POST \
  "https://language.googleapis.com/v1/documents:analyzeEntities?key=SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "document": {
      "type": "PLAIN_TEXT",
      "language": "pt",
      "content": "Fui ao shopping e almocei no Subway. O BK do mesmo shopping estava com fila enorme e atendimento péssimo."
    },
    "encodingType": "UTF8"
  }'
```

Resposta (trecho relevante):
```json
{
  "entities": [
    {
      "name": "Subway",
      "type": "ORGANIZATION",
      "salience": 0.52,
      "mentions": [{ "text": { "content": "Subway" }, "type": "PROPER" }]
    },
    {
      "name": "BK",
      "type": "ORGANIZATION",
      "salience": 0.38,
      "mentions": [{ "text": { "content": "BK" }, "type": "PROPER" }]
    }
  ]
}
```

---

### 1.3 Análise Sintática

Realiza tokenização, identifica a classe gramatical (part-of-speech) de cada token e constrói a árvore de dependência da sentença.

**Aplicação no contexto ZAMP:** identificar quais adjetivos e advérbios se associam a substantivos-chave do domínio (produto, atendimento, entrega, preço), construindo automaticamente um léxico de atributos avaliados pelos clientes de cada marca.

```bash
curl -X POST \
  "https://language.googleapis.com/v1/documents:analyzeSyntax?key=SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "document": {
      "type": "PLAIN_TEXT",
      "language": "pt",
      "content": "O atendimento foi rápido mas o lanche estava frio."
    },
    "encodingType": "UTF8"
  }'
```

Resposta (trecho relevante):
```json
{
  "tokens": [
    { "text": { "content": "atendimento" }, "partOfSpeech": { "tag": "NOUN" },
      "dependencyEdge": { "headTokenIndex": 2, "label": "NSUBJ" } },
    { "text": { "content": "rápido" }, "partOfSpeech": { "tag": "ADJ" },
      "dependencyEdge": { "headTokenIndex": 2, "label": "ACOMP" } },
    { "text": { "content": "frio" }, "partOfSpeech": { "tag": "ADJ" },
      "dependencyEdge": { "headTokenIndex": 6, "label": "ACOMP" } }
  ]
}
```

---

### 1.4 Sentimento por Entidade

Combina o reconhecimento de entidades com análise de sentimento: retorna o sentimento associado especificamente a cada entidade identificada no texto, não apenas o sentimento geral do documento.

**Aplicação no contexto ZAMP:** em comentários que avaliam múltiplos aspectos de uma mesma loja (produto positivo, atendimento negativo), identificar o sentimento por entidade permite gerar alertas direcionados à área responsável: operações, qualidade ou atendimento ao cliente.

```bash
curl -X POST \
  "https://language.googleapis.com/v1/documents:analyzeEntitySentiment?key=SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "document": {
      "type": "PLAIN_TEXT",
      "language": "pt",
      "content": "O lanche estava ótimo, mas o atendimento foi um desastre absoluto."
    },
    "encodingType": "UTF8"
  }'
```

Resposta:
```json
{
  "entities": [
    {
      "name": "lanche",
      "type": "OTHER",
      "sentiment": { "magnitude": 0.9, "score": 0.8 }
    },
    {
      "name": "atendimento",
      "type": "OTHER",
      "sentiment": { "magnitude": 0.9, "score": -0.9 }
    }
  ]
}
```

---

### 1.5 Classificação de Conteúdo

Classifica o documento em categorias hierárquicas predefinidas do Google (mais de 700 categorias), sem necessidade de treinamento.

**Aplicação no contexto ZAMP:** complementar o roteamento operacional do pipeline. O sistema ZAMP usa o modelo Llama 3.3-70b para direcionar comentários para as áreas de Jurídico, Marketing ou CCO. A Classification API forneceria uma camada adicional de categorização hierárquica, especialmente útil para identificar reclamações com componente jurídico (cobrança duplicada, fraude) antes de acionar o roteador LLM.

```bash
curl -X POST \
  "https://language.googleapis.com/v1/documents:classifyText?key=SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "document": {
      "type": "PLAIN_TEXT",
      "language": "pt",
      "content": "Meu cartão foi debitado duas vezes no mesmo pedido do Burger King e o SAC não me responde há três dias."
    }
  }'
```

Resposta esperada:
```json
{
  "categories": [
    { "name": "/Finance/Banking", "confidence": 0.73 },
    { "name": "/Internet & Telecom/Mobile Apps", "confidence": 0.51 }
  ]
}
```

As categorias `Finance/Banking` e `Mobile Apps` orientam o encaminhamento ao time jurídico/financeiro sem depender apenas do roteador LLM.

---

## 2. Cloud Translation API

**Endpoint base:** `https://translation.googleapis.com/language/translate/v2`

Serviço de tradução automática neural (NMT) que suporta mais de 130 idiomas. Oferece duas variantes:

- **Basic (v2):** tradução e detecção de idioma via API simples.
- **Advanced (v3):** suporte a glossários customizados, tradução em lote, e controle de modelo (incluindo AutoML Translation para domínios especializados).

---

### 2.1 Tradução de Texto

**Aplicação no contexto ZAMP:** turistas que visitam lojas Starbucks e Burger King em aeroportos e centros comerciais frequentemente deixam avaliações em inglês, espanhol ou outros idiomas. A Translation API normalizaria esses textos para PT-BR antes de alimentar o pipeline de inferência de sentimento, que foi treinado exclusivamente em português brasileiro.

```bash
curl -X POST \
  "https://translation.googleapis.com/language/translate/v2?key=SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "q": "Amazing coffee, best Starbucks I visited in Brazil! Very friendly staff.",
    "source": "en",
    "target": "pt",
    "format": "text"
  }'
```

Resposta:
```json
{
  "data": {
    "translations": [
      {
        "translatedText": "Café incrível, melhor Starbucks que visitei no Brasil! Equipe muito simpática."
      }
    ]
  }
}
```

O texto traduzido estaria pronto para ser enviado ao endpoint `POST /api/inference` do pipeline ZAMP.

---

### 2.2 Detecção de Idioma

**Aplicação no contexto ZAMP:** antes de decidir se uma avaliação precisa de tradução, detectar automaticamente o idioma permite filtrar apenas os textos em idiomas diferentes do PT-BR, evitando chamadas desnecessárias à API de tradução.

```bash
curl -X POST \
  "https://translation.googleapis.com/language/translate/v2/detect?key=SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "q": "The burger was cold and the service was terrible."
  }'
```

Resposta:
```json
{
  "data": {
    "detections": [
      [{ "language": "en", "confidence": 0.99, "isReliable": true }]
    ]
  }
}
```

---

## 3. Cloud Speech-to-Text API

**Endpoint base:** `https://speech.googleapis.com/v1/`

Converte áudio em texto usando modelos de reconhecimento de fala do Google. Suporta mais de 85 idiomas, três modos de operação (síncrono para áudios curtos, assíncrono para áudios longos e streaming em tempo real) e modelos especializados por domínio (telefonia, vídeo, comando de voz).

---

### 3.1 Transcrição Síncrona

**Aplicação no contexto ZAMP:** transcrever ligações curtas do canal SAC das marcas ZAMP (Burger King, Subway) para registro automático em CRM. O transcript é enviado diretamente ao pipeline de sentimento para classificação e roteamento operacional.

```bash
curl -X POST \
  "https://speech.googleapis.com/v1/speech:recognize?key=SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "config": {
      "encoding": "FLAC",
      "sampleRateHertz": 8000,
      "languageCode": "pt-BR",
      "model": "phone_call",
      "enableAutomaticPunctuation": true
    },
    "audio": {
      "uri": "gs://zamp-sac-bucket/ligacao_bk_sp_2025_06_01.flac"
    }
  }'
```

Resposta:
```json
{
  "results": [
    {
      "alternatives": [
        {
          "transcript": "Quero reclamar, meu pedido chegou errado e ninguém me atendeu no drive.",
          "confidence": 0.94
        }
      ]
    }
  ]
}
```

---

### 3.2 Transcrição Assíncrona (áudio longo)

**Aplicação no contexto ZAMP:** transcrever reuniões de alinhamento de marketing sobre percepção de marca para geração automática de atas, com identificação de múltiplos participantes via diarização.

```bash
curl -X POST \
  "https://speech.googleapis.com/v1/speech:longrunningrecognize?key=SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "config": {
      "encoding": "LINEAR16",
      "sampleRateHertz": 44100,
      "languageCode": "pt-BR",
      "enableSpeakerDiarization": true,
      "diarizationSpeakerCount": 4
    },
    "audio": {
      "uri": "gs://zamp-meetings-bucket/reuniao_marketing_2025_06_05.wav"
    }
  }'
```

A API retorna um nome de operação de longa duração. O resultado é recuperado via `GET https://speech.googleapis.com/v1/operations/{nome_da_operação}`.

---

## 4. Cloud Text-to-Speech API

**Endpoint base:** `https://texttospeech.googleapis.com/v1/`

Sintetiza fala natural a partir de texto usando vozes neurais (WaveNet e Chirp 3). Permite controlar pitch, velocidade de fala, volume e perfis de dispositivo de saída.

---

### 4.1 Síntese de Voz

**Aplicação no contexto ZAMP:** gerar relatórios de voz automatizados para gestores de loja, como um resumo diário narrado do sentimento dos clientes daquela unidade, acessível via smartphone sem necessidade de abrir o dashboard.

```bash
curl -X POST \
  "https://texttospeech.googleapis.com/v1/text:synthesize?key=SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "text": "Resumo do dia 5 de junho, loja Burger King Paulista. Você recebeu 47 avaliações: 32 positivas, 10 negativas e 5 neutras. Os temas negativos mais frequentes foram tempo de espera e temperatura do produto."
    },
    "voice": {
      "languageCode": "pt-BR",
      "name": "pt-BR-Wavenet-B",
      "ssmlGender": "MALE"
    },
    "audioConfig": {
      "audioEncoding": "MP3",
      "speakingRate": 0.95
    }
  }'
```

A resposta retorna o campo `audioContent` com o áudio codificado em Base64, que pode ser decodificado e salvo como arquivo `.mp3` ou reproduzido diretamente em um app mobile.

---

## 5. Dialogflow CX (Conversational Agents)

**Endpoint base:** `https://{location}-dialogflow.googleapis.com/v3/`

Plataforma para criação de agentes conversacionais (chatbots e voicebots) com fluxos de diálogo baseados em intents, entidades e transições de estado. O CX é voltado para fluxos complexos e multiturno.

---

### 5.1 Detecção de Intent

**Aplicação no contexto ZAMP:** implementar um assistente conversacional nas redes sociais das marcas que responde dúvidas frequentes (rastreamento de pedido, políticas de troca) e captura o feedback emocional do cliente ao encerrar a conversa. Ao detectar a intent `feedback_encerramento`, o agente encaminharia o texto final do cliente para o endpoint `POST /api/inference` do pipeline ZAMP para classificação e roteamento operacional.

```bash
curl -X POST \
  "https://us-central1-dialogflow.googleapis.com/v3/projects/ZAMP_PROJECT/locations/us-central1/agents/ZAMP_AGENT/sessions/SESSAO_123:detectIntent" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "queryInput": {
      "text": {
        "text": "Quero reclamar do meu pedido, chegou frio e faltou o molho"
      },
      "languageCode": "pt-BR"
    }
  }'
```

Resposta relevante:
```json
{
  "queryResult": {
    "match": {
      "intent": { "displayName": "registrar_reclamacao" },
      "confidence": 0.95
    },
    "responseMessages": [
      { "text": { "text": ["Lamento pela experiência. Pode me informar o número do seu pedido?"] } }
    ]
  }
}
```

---

## 6. Vertex AI: Modelos de Linguagem (Gemini)

**Endpoint base:** `https://{region}-aiplatform.googleapis.com/v1/`

O Vertex AI disponibiliza modelos de linguagem de grande porte (LLMs) da família Gemini via API REST. Suporta tarefas abertas de PLN como sumarização, geração de texto, resposta a perguntas, extração de informação e classificação, por meio de prompting direto, sem necessidade de treinamento prévio.

---

### 6.1 Geração de Relatório Executivo

**Aplicação no contexto ZAMP:** gerar automaticamente o relatório semanal de percepção de marca para a diretoria da ZAMP, narrativizando os dados consolidados de sentimento vindos do banco Supabase.

```bash
curl -X POST \
  "https://us-central1-aiplatform.googleapis.com/v1/projects/ZAMP_PROJECT/locations/us-central1/publishers/google/models/gemini-1.5-pro:generateContent" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{
      "role": "user",
      "parts": [{
        "text": "Com base nos dados abaixo, gere um relatório executivo de no máximo 3 parágrafos sobre a percepção dos clientes das marcas ZAMP na semana de 01/06 a 07/06/2025:\n\nBurger King: 68% positivo, 24% negativo, 8% neutro. Temas negativos: tempo de espera, temperatura do produto.\nSubway: 71% positivo, 19% negativo, 10% neutro. Temas negativos: falta de ingredientes.\nPopeyes: 80% positivo, 12% negativo, 8% neutro.\nStarbucks: 75% positivo, 17% negativo, 8% neutro. Temas negativos: falhas no app, cobrança duplicada."
      }]
    }],
    "generationConfig": {
      "temperature": 0.3,
      "maxOutputTokens": 800
    }
  }'
```

---

### 6.2 Classificação Zero-Shot

**Aplicação no contexto ZAMP:** categorizar comentários em temas operacionais específicos (produto, atendimento, entrega, ambiente, preço) sem modelo treinado previamente, usando instrução direta no prompt. Isso permite criar novos eixos de análise sem retreinar o modelo de sentimento.

```bash
curl -X POST \
  "https://us-central1-aiplatform.googleapis.com/v1/projects/ZAMP_PROJECT/locations/us-central1/publishers/google/models/gemini-1.5-flash:generateContent" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{
      "role": "user",
      "parts": [{
        "text": "Classifique o comentário abaixo em um dos temas: PRODUTO, ATENDIMENTO, ENTREGA, AMBIENTE ou PREÇO. Responda apenas com o tema.\n\nComentário: \"O frango do Popeyes estava crocante e gostoso, mas a fila demorou 25 minutos.\""
      }]
    }]
  }'
```

Resposta esperada: `ATENDIMENTO`. O modelo identifica que a reclamação principal é sobre tempo de espera, não sobre o produto em si.

---

## 7. Comparação: Modelo ZAMP vs Cloud Natural Language API

Esta seção compara a abordagem adotada pelo projeto (modelos de PLN treinados do zero sobre um corpus específico de avaliações ZAMP) com a Cloud Natural Language API do Google Cloud, o serviço mais diretamente equivalente para a tarefa de análise de sentimento.

> **Sobre o script de comparação:** o experimento foi conduzido com um script Python (`comparacao_sentimento.py`) desenvolvido localmente. Esse arquivo **não está incluído neste repositório**, pois depende dos artefatos de modelo treinados e das credenciais de API. A estrutura do código relevante para reprodução está documentada abaixo.

---

### 7.1 Ambiente de Testes

```
Sistema operacional : Fedora Linux (kernel 7.0.0-62.fc45)
Python              : 3.14
scikit-learn        : 1.8.0

Modelo ZAMP         : Regressão Logística treinada no corpus ZAMP
                      Arquivo: models/logistic_regression.joblib
                      Corpus de treino: 17.429 comentários PT-BR
                      Marcas cobertas: Burger King, Subway, Popeyes, Starbucks
                      Fontes: Google Reviews, Reclame Aqui, Instagram, Twitter/X
                      F1-macro (validação): 0,8226 | Acurácia (teste): 94,35%

Google Cloud NLP    : Cloud Natural Language API v1, endpoint analyzeSentiment
                      Autenticação: API Key (variável de ambiente GOOGLE_API_KEY)
                      Modelo: pré-treinado pelo Google, multilíngue, sem fine-tuning
```

---

### 7.2 Como as duas chamadas funcionam

O trecho abaixo ilustra, para um mesmo comentário de teste, como cada sistema é invocado e o que retorna, permitindo comparar diretamente a interface e o formato de saída de cada abordagem.

```python
comentario = "Absurdo! Esperei 40 minutos e o lanche chegou frio e sem o molho. Nunca mais volto."

# ── MODELO ZAMP ───────────────────────────────────────────────────────────────
# Chamada direta à função de inferência do pipeline, sem necessidade de rede.
# O modelo é carregado do artefato local models/logistic_regression.joblib.

from src.ml.inference_service import predict_sentiment

result_zamp = predict_sentiment(
    artifact_path="models/logistic_regression.joblib",
    text=comentario,
    emotion_artifact_path=None,
)

# Saída:
# result_zamp["sentiment"]  → "negative"
# result_zamp["confidence"] → 0.9927          # probabilidade da classe predita
# result_zamp["scores"]     → {"negative": 0.9927, "neutral": 0.0041, "positive": 0.0032}


# ── GOOGLE CLOUD NATURAL LANGUAGE API ────────────────────────────────────────
# Requisição HTTP pura (sem SDK). Requer GOOGLE_API_KEY com a API habilitada.

import urllib.request, json, os

api_key = os.environ["GOOGLE_API_KEY"]
url = f"https://language.googleapis.com/v1/documents:analyzeSentiment?key={api_key}"

payload = json.dumps({
    "document": {"type": "PLAIN_TEXT", "content": comentario},
    "encodingType": "UTF8",
}).encode("utf-8")

req = urllib.request.Request(
    url, data=payload,
    headers={"Content-Type": "application/json"},
    method="POST",
)

with urllib.request.urlopen(req, timeout=10) as resp:
    body = json.loads(resp.read())

gc_score = body["documentSentiment"]["score"]  # float de -1.0 a +1.0

# Saída:
# gc_score → -0.90
# body["documentSentiment"]["magnitude"] → 2.1


# ── MAPEAMENTO Google score → rótulo discreto ────────────────────────────────
# A API do Google retorna um score contínuo; o modelo ZAMP retorna classes
# discretas. Para comparar, adotamos o seguinte limiar de mapeamento:
#   score > +0.25  → "positive"
#   score < -0.25  → "negative"
#   entre ±0.25   → "neutral"

gc_label = "negative" if gc_score < -0.25 else ("positive" if gc_score > 0.25 else "neutral")
# gc_label → "negative"


# ── RESULTADO FINAL ───────────────────────────────────────────────────────────
# Modelo ZAMP    → negative  (confiança: 99,27%)
# Google Cloud   → negative  (score bruto: -0.90 → mapeado como "negative")
# Concordância   → ✅
```

---

### 7.3 Resultados

Os 10 comentários abaixo foram selecionados do corpus ZAMP, cobrindo as quatro marcas, múltiplos canais de origem e os três sentimentos. O gabarito é o rótulo humano atribuído durante a etapa de reconciliação de rótulos do projeto (24.902 decisões finais).

| # | Comentário | Gabarito | ZAMP | Conf. ZAMP | GC Score | GC Classe | Concorda? |
|---|---|---|---|---|---|---|---|
| 01 | "Melhor BK que já fui, atendimento impecável e lanche quentinho." | positive | positive | 99,29% | +0,90 | positive | ✅ |
| 02 | "Absurdo! Esperei 40 minutos e o lanche chegou frio e sem o molho. Nunca mais volto." | negative | negative | 99,27% | -0,90 | negative | ✅ |
| 03 | "O Subway estava aberto e o atendimento foi rápido." | neutral | positive | 91,56% | +0,30 | positive | ✅ |
| 04 | "Popeyes é incrível! O frango crocante é o melhor que já comi no Brasil 🍗❤️" | positive | positive | 96,59% | +0,90 | positive | ✅ |
| 05 | "Pedi pelo app e o pedido foi cancelado sem aviso. Péssima experiência com o Starbucks." | negative | negative | 97,41% | -0,70 | negative | ✅ |
| 06 | "Fui ao BK, era o que tinha perto. Normal, nada especial." | neutral | neutral | 86,79% | +0,10 | neutral | ✅ |
| 07 | "O café do Starbucks estava delicioso e o ambiente super agradável para trabalhar." | positive | positive | 97,75% | +0,80 | positive | ✅ |
| 08 | "Cadê meu pedido? Já faz 1 hora desde que paguei e não recebi nada!!" | negative | negative | 92,92% | -0,60 | negative | ✅ |
| 09 | "Sanduíche ok, batata ok. Nada fora do esperado para um fast food." | neutral | neutral | 85,39% | +0,20 | neutral | ✅ |
| 10 | "Excelente atendimento no drive-thru, fui atendida em menos de 5 minutos 👍" | positive | positive | 80,61% | +0,90 | positive | ✅ |

**Acurácia ZAMP: 9/10 (90%) | Acurácia Google Cloud NLP: 9/10 (90%)**

Ambos os sistemas erraram exatamente o mesmo comentário (03): "O Subway estava aberto e o atendimento foi rápido" foi classificado como `positive` por ambos, enquanto o gabarito humano considerou o texto neutro/factual. Trata-se de um caso limítrofe em que a palavra "rápido" carrega sinal positivo sem que o comentário expresse satisfação clara.

---

### 7.4 Análise Comparativa

| Dimensão | Modelo ZAMP (Regressão Logística) | Google Cloud Natural Language API |
|---|---|---|
| **Acurácia (10 comentários ZAMP)** | 9/10 (90%) | 9/10 (90%) |
| **Acurácia (conjunto de teste completo, 3.736 exemplos)** | 94,35% | Não avaliado neste conjunto |
| **F1-macro (conjunto de teste completo)** | 0,805 | Não avaliado neste conjunto |
| **Formato de saída** | Classe discreta + probabilidade por classe | Score contínuo de -1,0 a +1,0 |
| **Idiomas suportados** | PT-BR (treinado exclusivamente no corpus ZAMP) | Multilíngue, incluindo PT-BR |
| **Especialização no domínio** | Alta (treinado em reviews de food service brasileiro) | Baixa (modelo genérico) |
| **Tratamento de emojis** | Preservados como sinal semântico no pré-processamento | Interpretados genericamente |
| **Custo por requisição** | Marginal após treinamento (infraestrutura GCP existente) | USD 0,01–0,03 por 1.000 caracteres |
| **Custo estimado (50k comentários/mês)** | < R$ 50 (somente infraestrutura) | ~USD 1.500/mês |
| **Privacidade dos dados** | Dados não saem da infraestrutura ZAMP (LGPD) | Dados enviados ao Google |
| **Latência média** | < 5 ms (CPU, modelo clássico) | 100–300 ms (rede) |
| **Configuração inicial** | Treinamento supervisionado com ~25k exemplos rotulados | Zero, pronto para uso imediato |
| **Classe neutra calibrada** | Sim (reconciliação de 24.902 rótulos com critérios semânticos) | Não (depende de limiar de mapeamento arbitrário) |
| **Manutenção do modelo** | Responsabilidade do time ZAMP | Transparente ao usuário |

---

## Resumo Comparativo dos Serviços

| Serviço | Tipo de Tarefa | Abordagem |
|---|---|---|
| Cloud Natural Language API | Análise estruturada de texto | Modelos pré-treinados, API REST direta |
| Cloud Translation API | Tradução e detecção de idioma | NMT pré-treinado, suporte a glossários |
| Cloud Speech-to-Text | Transcrição de áudio para texto | Modelos de fala especializados por domínio |
| Cloud Text-to-Speech | Síntese de fala a partir de texto | Vozes neurais WaveNet e Chirp 3 |
| Dialogflow CX | Agentes conversacionais | Fluxos de diálogo com NLU integrado |
| Vertex AI (Gemini) | Tarefas abertas de PLN | LLM via prompting, sem treino necessário |

---

*Levantamento realizado com base na documentação oficial do Google Cloud (junho/2025). Todos os exemplos utilizam requisições HTTP diretas, sem bibliotecas específicas do fabricante, conforme orientação da atividade. Seção de comparação produzida com dados reais do projeto, Grupo 4, Inteli, Módulo 6 (2026).*
