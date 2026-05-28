# phone-ai
挑选手机
import streamlit as st
from openai import OpenAI
import sqlite3
from PIL import Image
import os

# --- 配置 ---
client = OpenAI(
    api_key="sk-0b047a2ebdd14306936af6000bdcdc4f"
    base_url="https://api.deepseek.com"
)

# 数据库路径
DB_PATH = "phones.db"
# 图片存储文件夹
IMAGE_DIR = "phone_images"

# --- 从数据库查图片路径 ---
def get_phone_image(brand, model, version, color, condition):
    """根据手机信息，从数据库查到对应的图片路径"""
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute("""
        SELECT image_path FROM phones 
        WHERE brand=? AND model=? AND version=? AND color=? AND condition=?
    """, (brand, model, version, color, condition))
    result = cursor.fetchone()
    conn.close()
    if result and result[0]:
        return result[0]
    return None

# --- AI推荐（稍作调整，让AI返回结构化信息） ---
def pick_phone(user_input):
    response = client.chat.completions.create(
        model="deepseek-v4-pro",
        messages=[
            {"role": "system", "content": """你是二手手机批发顾问。根据用户需求，推荐3款手机。
            返回格式（严格按此格式，每款手机一行）：
            品牌 | 型号 | 版本 | 颜色 | 成色 | 价格 | 推荐理由
            例如：苹果 | iPhone 14 Pro | 256G | 紫色 | A级 | 5850 | 自拍强，适合女生"""},
            {"role": "user", "content": user_input}
        ],
        stream=False
    )
    return response.choices[0].message.content

# --- 解析AI返回的结构化数据 ---
def parse_recommendations(ai_response):
    """把AI返回的文本解析成列表"""
    lines = ai_response.strip().split("\n")
    phones = []
    for line in lines:
        if "|" in line:
            parts = [p.strip() for p in line.split("|")]
            if len(parts) >= 7:
                phones.append({
                    "brand": parts[0],
                    "model": parts[1],
                    "version": parts[2],
                    "color": parts[3],
                    "condition": parts[4],
                    "price": parts[5],
                    "reason": parts[6]
                })
    return phones

# --- Streamlit界面 ---
st.set_page_config(page_title="AI挑手机", page_icon="📱")
st.title("📱 二手手机AI助手 - 实拍验机")

# 聊天逻辑
if "messages" not in st.session_state:
    st.session_state.messages = []

for message in st.session_state.messages:
    with st.chat_message(message["role"]):
        # 如果是助手消息且包含图片，就展示图片
        if message["role"] == "assistant" and "images" in message:
            st.markdown(message["content"])
            for img_path in message["images"]:
                if os.path.exists(img_path):
                    st.image(Image.open(img_path), caption=os.path.basename(img_path), width=300)
                else:
                    st.caption(f"（图片暂缺：{img_path}）")
        else:
            st.markdown(message["content"])

# 接收用户输入
if prompt := st.chat_input("比如：2000左右，拍照好的安卓机，成色无所谓"):
    # 用户消息
    st.chat_message("user").markdown(prompt)
    st.session_state.messages.append({"role": "user", "content": prompt})

    # AI处理
    with st.chat_message("assistant"):
        with st.spinner("正在帮你找货..."):
            ai_response = pick_phone(prompt)
            phones = parse_recommendations(ai_response)

        # 组装展示内容
        display_text = "根据您的需求，为您找到以下机型：\n\n"
        images_to_show = []

        for i, phone in enumerate(phones):
            display_text += f"**{i+1}. {phone['brand']} {phone['model']} {phone['version']}**\n"
            display_text += f"   颜色：{phone['color']} | 成色：{phone['condition']} | 价格：{phone['price']}元\n"
            display_text += f"   推荐理由：{phone['reason']}\n\n"

            # 查对应的实拍图
            img_path = get_phone_image(
                phone['brand'], phone['model'], phone['version'],
                phone['color'], phone['condition']
            )
            if img_path:
                images_to_show.append(img_path)

        st.markdown(display_text)

        # 展示图片
        for img_path in images_to_show:
            if os.path.exists(img_path):
                st.image(Image.open(img_path), caption=os.path.basename(img_path), width=300)
            else:
                st.caption(f"（图片暂缺：{img_path}）")

    # 保存消息（包含图片路径）
    st.session_state.messages.append({
        "role": "assistant",
        "content": display_text,
        "images": images_to_show
    })
