# bibliotecas instaladas no código

!pip install -q gradio requests
!pip install -q google-generativeai
!pip install -q pypdf python-docx reportlab

# Imports

from google.colab import userdata
import gradio as gr
import requests
import json
from pypdf import PdfReader
from docx import Document

# Inicio do código

from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer
from reportlab.lib.styles import getSampleStyleSheet

API_KEY = userdata.get("API_KEY")

if not API_KEY:
    raise ValueError("API_KEY não encontrada nos Secrets do Colab.")

URL = (
    f"https://generativelanguage.googleapis.com/v1beta/models/"
    f"gemini-2.5-flash:generateContent?key={API_KEY}"
)

# =====================================
# CONSULTA GEMINI
# =====================================

def consultar_gemini(prompt):

    headers = {
        "Content-Type": "application/json"
    }

    data = {
        "contents": [
            {
                "parts": [
                    {
                        "text": prompt
                    }
                ]
            }
        ]
    }

    try:

        response = requests.post(
            URL,
            headers=headers,
            data=json.dumps(data)
        )

        if response.status_code != 200:
            return f"Erro da API ({response.status_code}):\n{response.text}"

        resultado = response.json()

        return resultado["candidates"][0]["content"]["parts"][0]["text"]

    except Exception as erro:
        return f"Erro: {erro}"

# =====================================
# LEITURA DE ARQUIVOS
# =====================================

def ler_curriculo(arquivo): # Fazendo a leitura do curriculo do usuário

    texto = ""

    try:

        # TXT
        if arquivo.name.endswith(".txt"): # Verificando se o arquivo é TXT

            with open(arquivo.name, "r", encoding="utf-8") as f: # Abrindo o Arquivo e lendo
                texto = f.read()

        # PDF
        elif arquivo.name.endswith(".pdf"): # Verificando se o arquivo é PDF

            leitor = PdfReader(arquivo.name) # Lendo o PDF

            for pagina in leitor.pages: # Percorrendo todas as páginas.

                conteudo = pagina.extract_text() # Fazendo a extração do texto

                if conteudo:
                    texto += conteudo + "\n"

        # DOCX
        elif arquivo.name.endswith(".docx"): # Verificando o Word

            doc = Document(arquivo.name) # Abre o documento

            for paragrafo in doc.paragraphs: # Lê cada parágrafo.
                texto += paragrafo.text + "\n"

        else:
            return "Formato de arquivo não suportado."

        return texto

    except Exception as erro:
        return f"Erro ao ler arquivo: {erro}"

# =====================================
# GERADOR DE PDF
# =====================================

def gerar_pdf(texto): # Fazendo a criação do PDF melhorado

    caminho_pdf = "curriculo_melhorado.pdf"

    doc = SimpleDocTemplate(caminho_pdf) # Faz a criação do PDF

    estilos = getSampleStyleSheet() # Carrega estilos prontos

    conteudo = []

    conteudo.append(
        Paragraph("Currículo Melhorado por IA", estilos["Title"]) # Adiciona texto ao PDF.
    )

    conteudo.append(Spacer(1, 12)) # Cria espaço em branco

    for linha in texto.split("\n"): # Faz quebra de texto onde existe ENTER

        linha = linha.strip() # Remove espaços e ENTERs desnecessários.

        if linha:
            conteudo.append(
                Paragraph(
                    linha.replace("&", "&amp;"),
                    estilos["BodyText"]
                )
            )

    doc.build(conteudo) # Construindo o PDF

    return caminho_pdf # faz o retorno do arquivo gerado

# =====================================
# ANALISTA DE CURRÍCULOS
# =====================================

def especialista(curriculo_informado, arquivo): # Faz o recebimento do texto digitado e arquivo enviado

    texto_curriculo = curriculo_informado

    # Se enviou arquivo, ele tem prioridade
    if arquivo is not None:

        texto_lido = ler_curriculo(arquivo) # Extrai o conteúdo.

        if texto_lido.startswith("Erro"):
            return texto_lido, None

        texto_curriculo = texto_lido # Substitui o texto digitado pelo conteúdo do arquivo.

    if not texto_curriculo.strip():
        return "Por favor, cole um currículo ou envie um arquivo.", None

# =================================
# AVALIAÇÃO
# =================================

    prompt_avaliacao = f"""
Você é um especialista em recrutamento e seleção.

Analise o currículo abaixo:

{texto_curriculo}

Retorne obrigatoriamente:

1. Resumo profissional
2. Pontos fortes
3. Pontos fracos
4. Sugestões de melhoria
5. Nota de 0 a 10
6. Feedback final
"""

    avaliacao = consultar_gemini(prompt_avaliacao) # Envia o prompt para a IA.

# =================================
# CURRÍCULO MELHORADO
# =================================

    prompt_melhoria = f"""
Você é um especialista em elaboração de currículos.

Reescreva completamente o currículo abaixo.

Melhore:

- Organização
- Clareza
- Ortografia
- Estrutura
- Linguagem profissional

Monte o currículo utilizando a estrutura:

DADOS PESSOAIS

OBJETIVO PROFISSIONAL

FORMAÇÃO ACADÊMICA

EXPERIÊNCIA PROFISSIONAL

CURSOS E CERTIFICAÇÕES

COMPETÊNCIAS

Currículo original:

{texto_curriculo}
"""

    curriculo_melhorado = consultar_gemini(prompt_melhoria) # Recebe o currículo melhorado.

    arquivo_pdf = gerar_pdf(curriculo_melhorado) # Transforma o currículo melhorado em PDF.

    return avaliacao, arquivo_pdf # Retorna avaliação e PDF

# =====================================
# INTERFACE
# =====================================

with gr.Blocks(title="Analista de Currículos IA") as app:

    gr.Markdown("""
    <h1 style="text-align:center; font-size:60px;">
    📄 Analista de Currículos IA
    </h1>

    <p style="text-align:center; font-size:18px;">
    Cole seu currículo ou envie um arquivo TXT, PDF ou DOCX para receber uma análise profissional e uma versão otimizada em PDF.
    </p>
    """)

    with gr.Tab("Currículo"):

        curriculo = gr.Textbox(
            label="Cole seu currículo",
            lines=7,
            max_lines=100,
            placeholder="Cole aqui o conteúdo do currículo..."
        )

        arquivo_curriculo = gr.File(
            label="📎 Enviar currículo",
            file_types=[".txt", ".pdf", ".docx"]
        )

        btn_avaliar = gr.Button("Avaliar Currículo")

        avaliacao = gr.Textbox(
            label="Avaliação do Currículo",
            lines=7,
            max_lines=100
        )

        curriculo_pdf = gr.File(
            label="📄 Baixar Currículo Melhorado (PDF)"
        )

        btn_avaliar.click(
            fn=especialista,
            inputs=[
                curriculo,
                arquivo_curriculo
            ],
            outputs=[
                avaliacao,
                curriculo_pdf
            ]
        )

app.launch(share=True)
