# 🤖 Automações para Gestão de Encontros Formativos (Google Apps Script)

Este repositório contém dois scripts criados no Google Apps Script (GAS) para automatizar a rotina de postagens de materiais e avisos no Google Classroom, integrando YouTube, Google Drive e Google Sheets. Tudo 100% gratuito e sem necessidade de plataformas de terceiros (como Make ou Zapier).

## 📌 O que essas automações fazem?

1. **Robô de Materiais (Pós-encontro):** Toda segunda-feira, verifica uma playlist do YouTube em busca da gravação do último encontro. Ao encontrar, atualiza a planilha de controle, busca os slides/PDFs complementares em uma pasta específica do Google Drive e cria uma postagem de "Material" no Google Classroom com tudo anexado.
2. **Robô de Avisos (Pré-encontro):** Toda quinta-feira, lê uma planilha de programação e posta automaticamente no mural (Stream) do Classroom um aviso com o tema, a data, o link de leitura prévia e o link do Google Meet do próximo encontro, marcando na planilha que o aviso já foi enviado.

---

## ⚙️ Pré-requisitos e Configuração Inicial

Para rodar esses scripts, você precisará criar um projeto no Google Apps Script vinculado à sua conta do Google:
1. Abra o seu Google Drive, clique em **Novo > Mais > Google Apps Script** (ou acesse `script.new`).
2. Dê um nome para o projeto (ex: "Automações Formação").
3. No menu lateral esquerdo, vá em **Serviços**, clique no **+** e adicione duas APIs:
   * **YouTube Data API v3**
   * **Google Classroom API**

---

## 🔎 Como encontrar seus IDs (Passo a Passo)

Para configurar o código, você precisará preencher alguns IDs. Veja como encontrar cada um deles:

* **ID da Planilha (Spreadsheet):** Abra a planilha no navegador. Copie o código longo na URL que fica entre `/d/` e `/edit`.
* **ID da Playlist do YouTube:** Abra a playlist. Copie o código na URL que fica logo após `list=`.
* **ID da Pasta do Google Drive:** Abra a pasta mãe onde ficam as subpastas das reuniões. Copie o código na URL logo após `folders/`.
* **ID da Turma e do Tópico (Classroom):** Atenção aqui! O link do navegador do Classroom mostra letras (ex: `ODQ...`), mas a automação precisa do **ID Numérico**. Para descobrir seus IDs numéricos, rode a função auxiliar abaixo no seu Apps Script e olhe o "Registro de Execução" (Log):

```javascript
// FERRAMENTA AUXILIAR: Descobrir IDs numéricos do Classroom
function descobrirMeusIDsDoClassroom() {
  const turmas = Classroom.Courses.list({pageSize: 10}).courses;
  turmas.forEach(turma => {
    Logger.log('🎓 TURMA: ' + turma.name + ' | 👉 ID: ' + turma.id);
    try {
      const topicos = Classroom.Courses.Topics.list(turma.id).topic;
      if (topicos) {
        topicos.forEach(topico => Logger.log('   📂 Tópico: ' + topico.name + ' | 👉 ID: ' + topico.topicId));
      }
    } catch(e) {}
  });
}
```

---

## 🚀 Automação 1: Postagem de Gravação e Materiais

**Como funciona a estrutura de pastas e nomenclatura:**
* Os vídeos no YouTube precisam ter "R" seguido do número (Ex: `R4`, `R10`).
* As subpastas no Drive devem seguir o padrão `00_Reunião 0` (Ex: `04_Reunião 4`).
* A planilha deve ter colunas saltando de 5 em 5 para os links das gravações (a partir da coluna 22).

**O Código:**
```javascript
function verificarPlaylistEPostar() {
  // 1. CONFIGURAÇÕES (Preencha com seus dados)
  const PLAYLIST_ID = 'SEU_ID_DA_PLAYLIST'; 
  const PLANILHA_ID = 'SEU_ID_DA_PLANILHA'; 
  const NOME_DA_ABA = 'NOME_DA_ABA'; 
  const LINHA_ALVO = 31; 
  
  const COURSE_ID = 'SEU_ID_NUMERICO_DA_TURMA';
  const TOPIC_ID = 'SEU_ID_NUMERICO_DO_TOPICO';
  const PASTA_MAE_ID = 'SEU_ID_DA_PASTA_NO_DRIVE';

  const planilha = SpreadsheetApp.openById(PLANILHA_ID);
  const aba = planilha.getSheetByName(NOME_DA_ABA);

  const resposta = YouTube.PlaylistItems.list('snippet', { playlistId: PLAYLIST_ID, maxResults: 20 });
  const videos = resposta.items;

  videos.forEach(video => {
    const titulo = video.snippet.title;
    const videoId = video.snippet.resourceId.videoId;
    const urlVideo = 'https://www.youtube.com/watch?v=' + videoId;
    const match = titulo.match(/R(\d+)/);

    if (match) {
      const numReuniao = parseInt(match[1], 10);
      if (numReuniao >= 4) {
        const colunaAlvo = 22 + (numReuniao - 1) * 5; 
        const celula = aba.getRange(LINHA_ALVO, colunaAlvo);
        
        // Se estiver vazio, é vídeo novo!
        if (celula.getValue() === '') {
          celula.setValue(urlVideo); 
          criarPostagemClassroom(numReuniao, urlVideo, COURSE_ID, TOPIC_ID, PASTA_MAE_ID);
        }
      }
    }
  });
}

function criarPostagemClassroom(numReuniao, urlVideo, courseId, topicId, pastaMaeId) {
  let numFormatado = numReuniao < 10 ? '0' + numReuniao : numReuniao;
  let nomePasta = numFormatado + '_Reunião ' + numReuniao;

  let pastaMae = DriveApp.getFolderById(pastaMaeId);
  let pastasEncontradas = pastaMae.getFoldersByName(nomePasta);
  if (!pastasEncontradas.hasNext()) return;

  let pastaReuniao = pastasEncontradas.next();
  let arquivos = pastaReuniao.getFiles();

  let materiais = [{ "link": { "url": urlVideo } }];

  while (arquivos.hasNext()) {
    let arquivo = arquivos.next();
    materiais.push({
      "driveFile": {
        "driveFile": { "id": arquivo.getId(), "title": arquivo.getName() },
        "shareMode": "VIEW" 
      }
    });
  }

  let postagem = {
    "title": "Materiais e Gravação - Reunião " + numReuniao,
    "description": "Olá! Confira em anexo os slides, materiais complementares e o link da gravação do nosso encontro formativo.",
    "topicId": topicId,
    "materials": materiais,
    "state": "PUBLISHED" // Troque para "DRAFT" se quiser testar antes
  };

  Classroom.Courses.CourseWorkMaterials.create(postagem, courseId);
}
```
**Agendamento (Acionador):** Configure um Trigger (ícone de Relógio) para rodar a função `verificarPlaylistEPostar` toda Segunda-feira, entre 18h e 19h.

---

## 📅 Automação 2: Avisos e Roteiros (Mural do Classroom)

**Preparação da Planilha:**
Crie uma aba específica com as seguintes colunas a partir de A1: `Reunião` | `Tema` | `Data` | `Link do Roteiro` | `Status`.

**O Código:**
```javascript
function enviarAvisoQuintaFeira() {
  const COURSE_ID = 'SEU_ID_NUMERICO_DA_TURMA'; 
  const PLANILHA_ID = 'SEU_ID_DA_PLANILHA'; 
  const NOME_ABA = 'NOME_DA_ABA_DOS_AVISOS'; 
  const LINK_MEET = 'https://meet.google.com/SEU-LINK-AQUI'; 
  
  const planilha = SpreadsheetApp.openById(PLANILHA_ID);
  const aba = planilha.getSheetByName(NOME_ABA);
  const dados = aba.getDataRange().getValues();
  
  for (let i = 1; i < dados.length; i++) {
    let reuniao = dados[i][0];
    let tema = dados[i][1];
    let dataEncontro = dados[i][2];
    let linkRoteiro = dados[i][3];
    let status = dados[i][4];
    
    let dataFormatada = dataEncontro;
    if (dataEncontro instanceof Date) {
      let dia = String(dataEncontro.getDate()).padStart(2, '0');
      let mes = String(dataEncontro.getMonth() + 1).padStart(2, '0');
      dataFormatada = dia + '/' + mes; 
    }
    
    if (status === "") {
      let mensagem = "Olá, pessoal!\n\n" +
                     "Passando para lembrar do nosso encontro da próxima segunda-feira, dia " + dataFormatada + " (" + reuniao + ").\n\n" +
                     "📚 O tema do nosso encontro será: " + tema + "\n\n" +
                     "📖 Segue o link do roteiro para leitura prévia:\n" + linkRoteiro + "\n\n" +
                     "🎥 Link do nosso Meet:\n" + LINK_MEET + "\n\n" +
                     "Até lá!";
      
      let anuncio = { "text": mensagem, "state": "PUBLISHED" };
      
      Classroom.Courses.Announcements.create(anuncio, COURSE_ID);
      aba.getRange(i + 1, 5).setValue("Enviado ✅");
      break; 
    }
  }
}
```
**Agendamento (Acionador):** Configure um Trigger para rodar a função `enviarAvisoQuintaFeira` toda Quinta-feira de manhã.

> 💡 **Dica de Ouro:** Se houver duas reuniões no mesmo dia (ex: R8 e R9), não altere o código! Preencha a planilha colocando tudo na mesma linha. Na coluna do link, cole o primeiro link, aperte `Alt + Enter` e cole o segundo link abaixo. O robô lerá perfeitamente!

---
