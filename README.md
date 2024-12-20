from flask import Flask, request, jsonify
import requests
from telegram import Bot
import threading
import time

# Configurações do Telegram
BOT_TOKEN = "SEU_TOKEN_TELEGRAM"
bot = Bot(token=BOT_TOKEN)

# Configurações da API de Futebol
API_URL = "https://api-football-v1.p.rapidapi.com/v3/"
API_HEADERS = {
    "x-rapidapi-key": "SUA_CHAVE_API",
    "x-rapidapi-host": "api-football-v1.p.rapidapi.com"
}

# Lista de bots configurados
bots = []

# Inicialização do Flask
app = Flask(__name__)

def buscar_partidas_ao_vivo():
    """
    Busca todas as partidas ao vivo.
    """
    endpoint = f"{API_URL}fixtures"
    params = {"live": "all"}  # Todas as partidas ao vivo
    response = requests.get(endpoint, headers=API_HEADERS, params=params)
    return response.json()

def monitorar_partidas(bot_config):
    """
    Monitora partidas ao vivo de acordo com as configurações do bot.
    """
    while bot_config.get("ativo", True):
        partidas = buscar_partidas_ao_vivo()
        for partida in partidas.get("response", []):
            estatisticas = partida.get("statistics", {})
            tempo = partida["fixture"]["status"].get("elapsed", 0)  # Tempo de jogo
            ataques_perigosos_casa = estatisticas.get("attacks_dangerous_home", 0)
            ataques_perigosos_visitante = estatisticas.get("attacks_dangerous_away", 0)

            ataques_totais = ataques_perigosos_casa + ataques_perigosos_visitante
            ataques_por_minuto = ataques_totais / tempo if tempo > 0 else 0

            if ataques_por_minuto >= bot_config["threshold"]:
                mensagem = (
                    f"⚽ Jogo em destaque: {partida['teams']['home']['name']} x {partida['teams']['away']['name']}\n"
                    f"⏰ Tempo: {tempo}'\n"
                    f"🔥 Ataques perigosos/minuto: {ataques_por_minuto:.2f}"
                )
                bot.send_message(chat_id=bot_config["chat_id"], text=mensagem)
        time.sleep(60)  # Verificar a cada minuto

@app.route('/create-bot', methods=['POST'])
def create_bot():
    bot_data = request.json
    bot_data["ativo"] = True
    bots.append(bot_data)
    threading.Thread(target=monitorar_partidas, args=(bot_data,)).start()
    return jsonify({"message": "Bot criado com sucesso!", "bot": bot_data})

@app.route('/bots', methods=['GET'])
def get_bots():
    return jsonify(bots)

@app.route('/bots/<int:bot_id>', methods=['DELETE'])
def delete_bot(bot_id):
    if 0 <= bot_id < len(bots):
        bots[bot_id]["ativo"] = False
        bots.pop(bot_id)
        return jsonify({"message": "Bot removido com sucesso!"})
    return jsonify({"error": "Bot não encontrado"}), 404

if __name__ == '__main__':
    app.run(debug=True)
