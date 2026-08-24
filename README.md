# Códigos Busca De Emprego N8N
Compilado dos códigos e comandos usados no video tutorial de n8n onde criamos um robô para buscar vagas no linkedin, filtrar de acordo com nossos requisitos e nos enviar diariamente no telegram, junto com um rascunho de carta de apresentação + link para aplicar para a vaga

## 1. Busca de vagas do LinkedIn com Apify:

Código:
```json
{
  "keywords": "Frontend Developer",
  "location": "Brazil"
}
```
<img width="390" height="636" alt="Captura de Tela 2026-08-19 às 16 08 12" src="https://github.com/user-attachments/assets/40e8e5be-8257-4a71-9ed9-b0255d09cd98" />

## 2. Filtro e otimização dos dados recebidos:

Código:
```js
// 1. Termos que PRECISAM estar no TÍTULO da vaga
const titulosFoco = [
  'frontend', 'front-end', 'front end',
  'fullstack', 'full-stack', 'full stack',
  'react', 'typescript', 'javascript', 'n8n'
];

// 2. Termos para descartar direto pelo TÍTULO
const termosDescarte = [
  'lead', 'tech lead', 'manager', 'gerente', 'director',
  'ios', 'android', 'flutter', 'qa', 'tester', 'data', 'dados', 
  'devops', 'c#', '.net', 'java', 'php', 'python', 'ruby'
];

const vagasFiltradas = $input.all().filter(item => {
  const titulo = (item.json.title || '').toLowerCase();

  // Verifica se o título tem pelo menos uma das suas stacks principais
  const tituloValido = titulosFoco.some(termo => titulo.includes(termo));
  
  // Verifica se é uma vaga de outra área/linguagem
  const ehDescartado = termosDescarte.some(termo => titulo.includes(termo));

  return tituloValido && !ehDescartado;
});

// Limita vagas para garantir zero estouro na API da Groq
return vagasFiltradas.slice(0, 10).map(item => ({
  json: {
    ...item.json,
    // Corta a descrição em 600 caracteres para economizar tokens
    description: (item.json.description || '').slice(0, 600)
  }
}));
```

<img width="625" height="671" alt="Captura de Tela 2026-08-19 às 16 10 45" src="https://github.com/user-attachments/assets/ca8bbda9-8058-4aac-bc7b-2f493b58326e" /> 

## 3. Análise e compatibilidade de vagas com IA:

Código:

```
Você é um recrutador técnico especialista. Analise a aderência desta vaga ao perfil do candidato.

PERFIL DO CANDIDATO:
- Função: Desenvolvedor(a) Fullstack / Frontend
- Stacks: JavaScript, TypeScript, React, Node.js, Docker, n8n.
- Preferência: Trabalho Remoto.

DADOS DA VAGA:
Vaga: {{ $json.title }}
Empresa: {{ $json.companyName }}
Local: {{ $json.location }}
Descrição: {{ ($json.description || '').slice(0, 1500) }}

INSTRUÇÕES DE RESPOSTA:
Responda EXCLUSIVAMENTE em formato JSON puro com as chaves:
{
  "score": 85,
  "pontos_fortes": ["..."],
  "requisitos_faltantes": ["..."],
  "resumo_vaga": "..."
}
```

<img width="402" height="486" alt="Captura de Tela 2026-08-19 às 16 17 18" src="https://github.com/user-attachments/assets/dee15832-f295-4651-99e7-2ba8bd530d28" /> <img width="402" height="451" alt="Captura de Tela 2026-08-19 às 16 13 40" src="https://github.com/user-attachments/assets/16323a41-4212-445d-9827-89146f47a026" /><img width="410" height="293" alt="Captura de Tela 2026-08-19 às 16 14 37" src="https://github.com/user-attachments/assets/d66ed1d4-b071-4f65-9df7-97b6487613c2" />

## 4. Transformar texto em dados estruturados:

Código:
```js
return $input.all().map((item, index) => {
  // 1. Pega o texto gerado pelo Basic LLM Chain
  let rawText = item.json.text || '';

  // Remove eventuais blocos de código markdown (ex: ```json ... ```)
  rawText = rawText.replace(/```json/gi, '').replace(/```/g, '').trim();

  // 2. Converte a string da IA em JSON
  let aiAnalysis = {};
  try {
    aiAnalysis = JSON.parse(rawText);
  } catch (error) {
    aiAnalysis = {
      score: 0,
      pontos_fortes: [],
      requisitos_faltantes: [],
      resumo_vaga: 'Erro ao processar JSON da IA'
    };
  }

  // 3. Resgata os dados originais da vaga pelo mesmo índice
  const job = $('Code in JavaScript').all()[index]?.json || {};

  // 4. Une tudo em um único objeto limpo
  return {
    json: {
      jobId: job.jobId || index,
      title: job.title || 'Título não informado',
      company: job.companyName || 'Empresa não informada',
      location: job.location || '',
      url: job.link || '',
      description: job.description || '',
      score: aiAnalysis.score || 0,
      pontos_fortes: aiAnalysis.pontos_fortes || [],
      requisitos_faltantes: aiAnalysis.requisitos_faltantes || [],
      resumo_vaga: aiAnalysis.resumo_vaga || ''
    }
  };
});
```
<img width="632" height="609" alt="Captura de Tela 2026-08-19 às 16 15 20" src="https://github.com/user-attachments/assets/344edcc4-2cf0-44f4-a1c9-60b941d31fc8" />

## 5. Condicional de compatibilidade

Código:
```
{{ $json.score }}
```

<img width="675" height="299" alt="Captura de Tela 2026-08-19 às 16 20 43" src="https://github.com/user-attachments/assets/e276b2d9-d623-4c7a-b111-d3837832316e" />

## 6. Carta de apresentação para cada vaga com IA:

Código User Message:

```
Vaga: {{ $json.title }} na empresa {{ $json.company }}
Resumo da vaga: {{ $json.resumo_vaga }}
Pontos fortes do meu perfil para essa vaga: {{ Array.isArray($json.pontos_fortes) ? $json.pontos_fortes.join(', ') : $json.pontos_fortes }}
```

Código Chat Message:

```
Você é um profissional sênior de tecnologia.
Escreva uma carta de apresentação em português, curta e direta para se candidatar à vaga.
DADOS DO CANDIDATO:
- Nome: Rafaella Ballerini
- Portfólio / GitHub: https://github.com/rafaballerini
O tom deve ser profissional e focado em resolver problemas. Assine com o nome do candidato ao final.
```

<img width="437" height="652" alt="Captura de Tela 2026-08-19 às 16 23 59" src="https://github.com/user-attachments/assets/355be97f-87d4-434e-8944-8bea3a6f4307" />


## 7 . Envio da mensagem pelo Telegram com informações das vagas selecionadas:

Código:

```
🎯 *NOVA VAGA COMPATÍVEL ENCONTRADA!*
📌 *Vaga:* {{ $('Code in JavaScript1').item.json.title }}
🏢 *Empresa:* {{ $('Code in JavaScript1').item.json.company }}
📍 *Local:* {{ $('Code in JavaScript1').item.json.location }}
📊 *Score de Aderência:* {{ $('Code in JavaScript1').item.json.score }}/100

💡 *Resumo:* _{{ $('Code in JavaScript1').item.json.resumo_vaga }}_

✅ *Pontos Fortes:
*• {{ Array.isArray($('Code in JavaScript1').item.json.pontos_fortes) ? $('Code in JavaScript1').item.json.pontos_fortes.join('\n• ') : $('Code in JavaScript1').item.json.pontos_fortes }}

✉️ *Sugestão de Carta de Apresentação:

*
{{ $json.text }}

🔗 [Clique aqui para abrir e aplicar na vaga]({{ $('Code in JavaScript1').item.json.url }})
```
<img width="413" height="249" alt="Captura de Tela 2026-08-19 às 16 24 47" src="https://github.com/user-attachments/assets/17e2c061-7c6d-46df-a492-34e9771d0cdb" />
