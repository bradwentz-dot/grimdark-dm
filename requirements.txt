import streamlit as st
import google.generativeai as genai
import re
import json
from docx import Document
import io

# Page Setup
st.set_page_config(page_title="Grimdark Solo DM", page_icon="🧙‍♂️", layout="wide")
st.title("🧙‍♂️ Grimdark Solo D&D Dungeon Master")

# 1. API Setup via Streamlit Secrets
if "GEMINI_API_KEY" in st.secrets:
    genai.configure(api_key=st.secrets["GEMINI_API_KEY"])
else:
    st.error("Please add your GEMINI_API_KEY in the Streamlit Cloud Secrets settings.")
    st.stop()

model = genai.GenerativeModel('gemini-2.5-flash')

# 2. Session State Memory
if "chat_history" not in st.session_state:
    st.session_state.chat_history = []
if "current_state" not in st.session_state:
    st.session_state.current_state = {}

# Helper function to generate Word document (.docx) in memory
def generate_docx(json_state):
    doc = Document()
    doc.add_heading('Current Campaign State', 0)
    formatted_state = json.dumps(json_state, indent=4)
    doc.add_paragraph(formatted_state)
    bio = io.BytesIO()
    doc.save(bio)
    return bio.getvalue()

# 3. Sidebar for State Tracker and Word Download
with st.sidebar:
    st.header("📊 Chronicle State Tracker")
    if st.session_state.current_state:
        st.json(st.session_state.current_state)
        
        # Download button for Word document (.docx)
        docx_data = generate_docx(st.session_state.current_state)
        st.download_button(
            label="📥 Download Campaign Log (.docx)",
            data=docx_data,
            file_name="character_sheet.docx",
            mime="application/vnd.openxmlformats-officedocument.wordprocessingml.document"
        )
    else:
        st.info("State updates will appear here once the story begins.")

# 4. Render Chat History
for message in st.session_state.chat_history:
    with st.chat_message(message["role"]):
        st.markdown(message["content"])

# 5. User Input
user_action = st.chat_input("What do you do?")

if user_action:
    # Render user prompt
    st.session_state.chat_history.append({"role": "user", "content": user_action})
    
    # Build prompt with invisible background state
    state_str = json.dumps(st.session_state.current_state) if st.session_state.current_state else "No prior state."
    full_prompt = f"Background State:\n{state_str}\n\nPlayer Action: {user_action}"

    # Query Gemini API
    with st.spinner("The Dungeon Master is writing your fate..."):
        response = model.generate_content(full_prompt)
        response_text = response.text

        # Extract <state> XML block
        state_match = re.search(r'<state>(.*?)</state>', response_text, re.DOTALL)
        if state_match:
            raw_json = state_match.group(1).strip()
            narrative_only = response_text.replace(state_match.group(0), "").strip()
            try:
                parsed_json = json.loads(raw_json)
                st.session_state.current_state = parsed_json
            except json.JSONDecodeError:
                pass
        else:
            narrative_only = response_text

        st.session_state.chat_history.append({"role": "assistant", "content": narrative_only})
        st.rerun()
