# AgentChinook
import os
from dataclasses import dataclass

from dotenv import load_dotenv
from langchain.tools import tool
from langchain_community.utilities import SQLDatabase
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from langchain_core.messages import HumanMessage

# ==================================================
# CARREGA VARIÁVEIS DE AMBIENTE
# ==================================================

load_dotenv()

# ==================================================
# BANCO DE DADOS
# ==================================================

db = SQLDatabase.from_uri("sqlite:///Chinook.db")

# ==================================================
# CARREGA O ESQUEMA DO BANCO
# ==================================================

schema = db.get_table_info(
    table_names=[
        "Track",
        "Album",
        "Artist",
        "Customer",
        "Invoice",
        "InvoiceLine",
        "Employee",
        "Genre"
    ]
)

# ==================================================
# FERRAMENTA SQL
# ==================================================

@tool
def execute_sql(query: str) -> str:
    """
    Executa consultas SQL em modo leitura.
    Utilize apenas SELECT.
    """
    try:
        print("\n==============================")
        print("SQL EXECUTADA")
        print("==============================")
        print(query)

        resultado = db.run(query)

        print("\n==============================")
        print("RESULTADO")
        print("==============================")
        print(resultado)

        return str(resultado)

    except Exception as e:
        return f"Error: {str(e)}"

# ==================================================
# PROMPT DO AGENTE
# ==================================================

SYSTEM_PROMPT = f"""
Você é um especialista em análise de dados SQLite.

Você está conectado ao banco Chinook.

Abaixo está o esquema do banco:

{schema}

REGRAS:

- Utilize a ferramenta execute_sql quando precisar consultar dados.
- Nunca invente tabelas.
- Nunca invente colunas.
- Utilize APENAS tabelas e colunas presentes no esquema acima.
- Gere apenas consultas SELECT.
- Nunca utilize:
    INSERT, UPDATE, DELETE, DROP, ALTER, CREATE, TRUNCATE

- Evite SELECT *.
- Se ocorrer erro SQL, revise a consulta utilizando o esquema acima.
- Limite os resultados a 5 linhas quando apropriado.
- Explique os resultados ao usuário em português.
"""

# ==================================================
# NVIDIA
# ==================================================

api_key = os.getenv("NVIDIA_API_KEY")

if not api_key:
    raise ValueError(
        "NVIDIA_API_KEY não encontrada. Verifique o arquivo .env"
    )

print("API NVIDIA carregada com sucesso!")

# ==================================================
# MODELO
# ==================================================

llm = ChatOpenAI(
    base_url="https://integrate.api.nvidia.com/v1",
    api_key=api_key,
    model="meta/llama-3.3-70b-instruct",
    temperature=0
)

# ==================================================
# AGENTE
# ==================================================

agent = create_react_agent(
    model=llm,
    tools=[execute_sql],
    prompt=SYSTEM_PROMPT
)

# ==================================================
# LOOP PRINCIPAL
# ==================================================

def main():

    print("=" * 60)
    print("AGENTE SQL CHINOOK + NVIDIA")
    print("Digite 'sair' para encerrar")
    print("=" * 60)

    while True:

        pergunta = input("\nPergunta: ")

        if pergunta.lower() == "sair":
            print("Encerrando...")
            break

        try:
            response = agent.invoke({
                "messages": [
                    HumanMessage(content=pergunta)
                ]
            })

            mensagens = response.get("messages", [])

            if mensagens:
                ultima_mensagem = mensagens[-1]

                print("\n====================================")
                print("RESPOSTA")
                print("====================================")

                if hasattr(ultima_mensagem, "content"):
                    print(ultima_mensagem.content)
                else:
                    print(ultima_mensagem)

            else:
                print(response)

        except Exception as erro:
            print("\nERRO:")
            print(erro)

# ==================================================
# EXECUÇÃO
# ==================================================

if _name_ == "_main_":
    main()****
