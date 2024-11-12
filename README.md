import cv2
import pytesseract
from PIL import Image
import re
import json
from google.colab import files

# Instalar o idioma português (por) para o Tesseract
!apt-get update
!apt-get install tesseract-ocr-por

# Configurar o caminho do Tesseract (em Colab, o caminho é geralmente este)
pytesseract.pytesseract.tesseract_cmd = "/usr/bin/tesseract"

# Função para tentar extrair os dados
def extract_data(text):
    # Expressões regulares para capturar os dados desejados
    data = {
        "Motivo da apreensão": None,
        "Cidade": None,
        "Pátio de apreensão": None,
        "Endereço do pátio": None,
        "Observações": None,
        "Órgão de apreensão": None
    }

    # Buscar o motivo da apreensão
    motivo_pattern = re.compile(r"(Motivo da apreensão|Motivo\s*[:\-]?\s*)([^\n]+)", re.IGNORECASE)
    match = motivo_pattern.search(text)
    if match:
        data["Motivo da apreensão"] = match.group(2).strip()

    # Buscar a cidade
    cidade_pattern = re.compile(r"(Cidade|Município)\s*[:\-]?\s*([^\n]+)", re.IGNORECASE)
    match = cidade_pattern.search(text)
    if match:
        data["Cidade"] = match.group(2).strip()

    # Buscar o pátio de apreensão
    patio_pattern = re.compile(r"(Pátio de apreensão|Pátio)\s*[:\-]?\s*([^\n]+)", re.IGNORECASE)
    match = patio_pattern.search(text)
    if match:
        data["Pátio de apreensão"] = match.group(2).strip()

    # Buscar o endereço do pátio
    endereco_pattern = re.compile(r"(Endereço do pátio|Endereço)\s*[:\-]?\s*([^\n]+)", re.IGNORECASE)
    match = endereco_pattern.search(text)
    if match:
        data["Endereço do pátio"] = match.group(2).strip()

    # Buscar as observações
    observacoes_pattern = re.compile(r"(Observações|Observação)\s*[:\-]?\s*([^\n]+)", re.IGNORECASE)
    match = observacoes_pattern.search(text)
    if match:
        data["Observações"] = match.group(2).strip()

    # Buscar o órgão de apreensão
    orgao_pattern = re.compile(r"(Órgão de apreensão|Órgão)\s*[:\-]?\s*([^\n]+)", re.IGNORECASE)
    match = orgao_pattern.search(text)
    if match:
        data["Órgão de apreensão"] = match.group(2).strip()

    return data

# Função principal
def process_image(image_path):
    # Carregar a imagem usando OpenCV
    imagem = cv2.imread(image_path)

    # Verificar se a imagem foi carregada corretamente
    if imagem is None:
        print(f"Erro: A imagem {image_path} não foi carregada corretamente.")
        return False, None

    # Converter a imagem do OpenCV (numpy array) para um objeto PIL
    pil_image = Image.fromarray(imagem)

    # Usar pytesseract para extrair texto da imagem
    texto = pytesseract.image_to_string(pil_image, lang='por')  # Definir o idioma como português

    # Exibir o texto extraído para depuração
    print("Texto extraído:", texto)

    # Verificar se o texto extraído contém dados suficientes
    if not texto.strip():
        return False, None  # Caso o OCR não consiga ler nada

    # Extrair os dados da string de texto
    extracted_data = extract_data(texto)

    # Exibir os dados extraídos para depuração
    print("Dados extraídos:", extracted_data)

    # Verificar se todos os dados essenciais foram extraídos corretamente
    if any(value is None for value in extracted_data.values()):
        print("Falha na extração de alguns dados.")
        return False, extracted_data  # Se faltar algum dado importante, retorna False

    # Salvar os dados em um arquivo JSON
    with open('dados_apreensao.json', 'w', encoding='utf-8') as f:
        json.dump(extracted_data, f, ensure_ascii=False, indent=4)

    return True, extracted_data

# Fazer upload da imagem
uploaded = files.upload()

# Verificar os arquivos carregados
for filename in uploaded.keys():
    print('Arquivo carregado:', filename)

# Definir o caminho da imagem carregada
image_path = list(uploaded.keys())[0]  # O nome do arquivo carregado será a chave do dicionário

# Processar a imagem
status, data = process_image(image_path)

# Imprimir o status e os dados extraídos
if status:
    print("Extração bem-sucedida. Dados:", data)
else:
    print("Falha na extração de dados.")
