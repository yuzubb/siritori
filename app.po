import os
import requests
import random
from flask import Flask, request, jsonify
# 形態素解析ライブラリ
from janome.tokenizer import Tokenizer
from janome.dic import Dictionary # 辞書データに直接アクセスするため

app = Flask(__name__)

# --- グローバル変数 ---
CHATWORK_API_TOKEN = os.environ.get("CHATWORK_API_TOKEN")
API_URL = "https://api.chatwork.com/v2"

# 状態管理用の辞書 {room_id: {"last_yomi": str, "used_words": set}}
GAME_STATUS = {}
# Bot自身のID
BOT_ACCOUNT_ID = None
# しりとり用単語リスト {先頭文字(ひらがな): {単語の読み(ひらがな)}}
SHIRITORI_WORDS = {}

# 形態素解析器の初期化
T = Tokenizer()

# --- 1. 初期化と単語リスト取得 ---

def initialize_bot():
    """BotのChatwork IDを取得し、Janome辞書から単語リストを生成する"""
    global BOT_ACCOUNT_ID
    
    if not CHATWORK_API_TOKEN:
        print("エラー: CHATWORK_API_TOKEN が設定されていません。")
        return

    # 1-A. Bot自身のIDを取得
    headers = {"X-ChatWorkToken": CHATWORK_API_TOKEN}
    try:
        response = requests.get(f"{API_URL}/me", headers=headers)
        response.raise_for_status()
        BOT_ACCOUNT_ID = response.json().get("account_id")
        print(f"✅ BotのChatwork IDを取得しました: {BOT_ACCOUNT_ID}")
    except Exception as e:
        print(f"❌ Bot IDの取得に失敗しました: {e}")

    # 1-B. 【単語リストの自動取得】
    print("⏳ Janome辞書からしりとり用単語リストを構築中...")
    dict_data = Dictionary().dic_data.items()
    
    # ひらがなへの変換マップ (カタカナ→ひらがな)
    KANA_MAP = {chr(i): chr(i - 96) for i in range(ord('ァ'), ord('ン') + 1)}

    for details in dict_data:
        surface = details[0]
        # 辞書構造に依存するが、ここでは簡易的に読みを取得
        if len(details[1]) > 7:
            reading = details[1][7]
        else:
            continue

        # 品詞チェック (名詞、動詞、形容詞など)
        if details[1][0].startswith(('名詞', '動詞', '形容詞')):
            
            if reading and reading != '*':
                # カタカナをひらがなに変換
                hiragana_yomi = "".join([KANA_MAP.get(c, c) for c in reading]).lower()
                
                # 「ん」で終わる単語は除外
                if not hiragana_yomi or hiragana_yomi[-1] == 'ん' or len(hiragana_yomi) < 2:
                    continue
                
                first_char = hiragana_yomi[0]
                
                if first_char not in SHIRITORI_WORDS:
                    SHIRITORI_WORDS[first_char] = set()
                
                # 辞書には「ひらがなの読み」を格納
                SHIRITORI_WORDS[first_char].add(hiragana_yomi)

    print(f"✅ 単語リスト構築完了。先頭文字の種類: {len(SHIRITORI_WORDS)}種類")


# --- 2. ユーティリティ関数 ---

def get_yomi(text):
    """形態素解析を行い、単語の正確な読み（ひらがな）を取得する"""
    yomi = ""
    # 複数単語が投稿された場合、Botは最初に見つけた単語の読みを採用する
    for token in T.tokenize(text):
        if token.part_of_speech.startswith(('名詞', '動詞', '形容詞')):
            reading = token.reading
            if reading and reading != '*':
                # カタカナをひらがなに変換
                yomi = "".join([chr(ord(c) - 96) if 'ァ' <= c <= 'ン' else c for c in reading])
                return yomi.lower()
            
    return ""

def get_surface_form(yomi):
    """ひらがなの読みから、対応するカタカナの表層形（Botが投稿する単語）を探す (簡易版)"""
    KANA_MAP = {chr(i): chr(i + 96) for i in range(ord('ぁ'), ord('ん') + 1)}
    return "".join([KANA_MAP.get(c, c) for c in yomi]).upper()

def get_last_char(yomi):
    """しりとりのルールに基づき、単語の末尾文字を取得する"""
    if not yomi:
        return None
    
    last_char = yomi[-1]
    
    # 伸ばし棒 ('ー') の処理
    if last_char == 'ー' and len(yomi) > 1:
        return yomi[-2]
    
    return last_char

def get_bot_word(next_char, used_words_set):
    """Botの単語リストから、次の文字で始まる、未使用の単語をランダムに選ぶ"""
    if next_char not in SHIRITORI_WORDS:
        return None, None 

    available_words = SHIRITORI_WORDS[next_char] - used_words_set
    
    if not available_words:
        return None, None 
    
    # ランダムに一つ選ぶ
    bot_yomi = random.choice(list(available_words))
    bot_surface = get_surface_form(bot_yomi)
    
    return bot_yomi, bot_surface

def send_chatwork_message(room_id, message_body):
    """Chatwork APIでメッセージを投稿する"""
    headers = {"X-ChatWorkToken": CHATWORK_API_TOKEN}
    data = {"body": message_body}
    
    try:
        response = requests.post(f"{API_URL}/rooms/{room_id}/messages", headers=headers, data=data)
        response.raise_for_status()
        print(f"✅ メッセージ送信成功 (Room: {room_id})")
    except Exception as e:
        print(f"❌ メッセージ送信失敗: {e}")

# --- 3. Webhookエンドポイント ---

@app.route("/webhook", methods=["POST"])
def chatwork_webhook():
    """ChatworkからのWebhookを受け取り、しりとりロジックを実行する"""
    
    # データのチェックと抽出
    if not request.is_json: return jsonify({"message": "Invalid format"}), 400
    event_data = request.json.get("webhook_event")
    if not event_data or event_data.get("body") is None: return jsonify({"message": "No event data"}), 200

    room_id = event_data.get("room_id")
    account_id = event_data.get("account_id")
    raw_body = event_data.get("body")
    
    # 0. Bot自身の投稿ならスキップ
    if account_id == BOT_ACCOUNT_ID:
        return jsonify({"message": "Self-message skipped"}), 200

    # 状態の初期化
    if room_id not in GAME_STATUS:
        GAME_STATUS[room_id] = {"last_yomi": None, "used_words": set()}
        
    status = GAME_STATUS[room_id]
    
    # --- コマンド判定 ---
    if 'しりとりスタート' in raw_body:
        status["last_yomi"] = "んご" # 初期単語の末尾が「ご」になるように設定（例）
        status["used_words"] = set()
        message = f"🎉 しりとりスタート！ [To:{account_id}] さん、好きな単語から始めてね！"
        send_chatwork_message(room_id, message)
        return jsonify({"message": "Game started"}), 200

    # ゲームが開始されていない場合
    if not status["last_yomi"]:
        message = f"[To:{account_id}] さん、しりとりを始めるには「しりとりスタート」と投稿してね！"
        send_chatwork_message(room_id, message)
        return jsonify({"message": "Waiting for start"}), 200

    # --- ユーザーのターン処理 ---
    
    # 1. 読みの取得と整形
    user_yomi = get_yomi(raw_body.strip())
    last_char = get_last_char(status["last_yomi"])
    
    # 2. ルール判定
    
    if not user_yomi:
        send_chatwork_message(room_id, f"[To:{account_id}] さん、単語が認識できませんでした。別の言葉で試してね。")
        return jsonify({"message": "No word recognized"}), 200
    
    if user_yomi in status["used_words"]:
        send_chatwork_message(room_id, f"[To:{account_id}] さん、それはもう使った単語 ({user_yomi}) ですよ！他の単語を言ってね。")
        return jsonify({"message": "Word already used"}), 200

    if get_last_char(user_yomi) == 'ん':
        # 3. エラー応答 ('ん'終了)
        send_chatwork_message(room_id, f"[To:{account_id}] さん、残念！「ん」で終わってしまいました。あなたの負けです...😭")
        status["last_yomi"] = None
        return jsonify({"message": "Game Over (N)"}), 200
    
    if user_yomi[0] != last_char:
        # 3. エラー応答 (文字違い)
        message = f"[To:{account_id}] さん、ルール違反です！前の単語は「{status['last_yomi']}」で「{last_char}」から始まります。「{last_char}」から始まる単語を言ってね。"
        send_chatwork_message(room_id, message)
        return jsonify({"message": "Rule violation"}), 200

    
    # --- 4. Botの応答（成功時） ---
    
    # 5-A. 状態の更新（ユーザーの単語をリストに追加）
    status["used_words"].add(user_yomi)
    
    # Botの次の単語を決定
    next_char = get_last_char(user_yomi)
    bot_yomi, bot_surface = get_bot_word(next_char, status["used_words"])

    if not bot_yomi:
        # Botの負け
        message = f"[To:{account_id}] さん、すごい！「{user_yomi}」の次の単語が見つかりませんでした... Botの負けです！🥳"
        status["last_yomi"] = None
        send_chatwork_message(room_id, message)
        return jsonify({"message": "User wins"}), 200

    # 5-B. 状態の更新（Botの単語をリストに追加し、最後の単語を更新）
    status["used_words"].add(bot_yomi)
    status["last_yomi"] = bot_yomi
    
    # 6. 結果の送信
    next_player_char = get_last_char(bot_yomi)
    
    response_message = f"""{bot_surface} ({bot_yomi})
[hr]
🎉 OKです！
次は [To:{account_id}] さん、あなたの番です！
「{next_player_char}」から始まる単語を言ってね。"""
    
    send_chatwork_message(room_id, response_message)
    
    return jsonify({"message": "Bot responded", "yomi": user_yomi}), 200

# --- サーバー起動 ---
if __name__ == "__main__":
    # Bot IDと単語リストを初期化
    initialize_bot()
    port = int(os.environ.get("PORT", 5000))
    app.run(host="0.0.0.0", port=port)
