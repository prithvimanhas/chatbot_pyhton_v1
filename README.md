mana version 1


import os
import uuid
import platform
import subprocess
from dotenv import load_dotenv
from fuzzywuzzy import fuzz
from textblob import TextBlob
import wikipedia
import spacy
import speech_recognition as sr
from gtts import gTTS
from pydub import AudioSegment
from pydub.playback import play

# ========================== INITIAL SETUP ==========================

# Load NLP
nlp = spacy.load("en_core_web_sm")

# Conversation memory
chat_history = []


# ========================== SPEECH FUNCTIONS ==========================

def speak(text: str):
    """Convert text to speech and play it."""
    filename = f"voice_{uuid.uuid4()}.mp3"
    tts = gTTS(text)
    tts.save(filename)

    audio = AudioSegment.from_file(filename, format="mp3")
    play(audio)
    os.remove(filename)


def listen() -> str:
    """Capture microphone input and return recognized text."""
    recognizer = sr.Recognizer()
    with sr.Microphone() as source:
        print("🎙 Listening...")
        audio = recognizer.listen(source)

    try:
        return recognizer.recognize_google(audio)
    except Exception as e:
        print(f"[ERROR - LISTEN]: {e}")
        return "Sorry, I couldn't understand."


# ========================== UTILITIES ==========================

def detect_sentiment(text: str) -> str:
    """Classify sentiment of text as positive, negative, or neutral."""
    polarity = TextBlob(text).sentiment.polarity
    if polarity > 0.2:
        return "positive"
    elif polarity < -0.2:
        return "negative"
    return "neutral"


def classify_intent(user_input: str):
    """Detect basic intent from user input."""
    intents = {
        "greeting": ["hi", "hello", "hey"],
        "goodbye": ["bye", "goodbye", "exit"],
        "name": ["your name", "who are you"]
    }
    for intent, phrases in intents.items():
        for phrase in phrases:
            if fuzz.partial_ratio(user_input.lower(), phrase) > 80:
                return intent
    return None


def handle_command(user_input: str):
    """Run simple system commands based on user request."""
    app_commands = {
        "notepad": "notepad",
        "calculator": "calc"
    }
    for app, command in app_commands.items():
        if f"open {app}" in user_input.lower():
            subprocess.Popen(["cmd", "/c", command])
            return f"Opening {app}..."
    return None


# ========================== RESPONSE SYSTEM ==========================

def fallback_response(user_input: str, user_name: str = "") -> str:
    """Generate a natural response without AI APIs."""

    # Save history
    chat_history.append(f"{user_name}: {user_input}")

    sentiment = detect_sentiment(user_input)
    intent = classify_intent(user_input)

    # Predefined intent responses
    if intent == "greeting":
        return f"Hello {user_name}! How are you feeling today?"
    elif intent == "goodbye":
        return "Goodbye! Have a wonderful day!"
    elif intent == "name":
        return "I’m Mana, your personal AI assistant."

    # Wikipedia lookup for factual queries
    try:
        summary = wikipedia.summary(user_input, sentences=2)
        return summary
    except Exception:
        pass

    # Sentiment-aware fallback
    if sentiment == "positive":
        return "That sounds great! Tell me more."
    elif sentiment == "negative":
        return "I’m sorry to hear that. Do you want to talk about it?"
    else:
        return "I’m here with you. What’s on your mind?"


# ========================== MAIN LOOP ==========================

def main():
    user_name = input("👋 Hi! What's your name? ")
    speak(f"Hello {user_name}, I'm Mana, your AI assistant.")

    while True:
        use_mic = input("🎤 Use mic? (yes/no): ").strip().lower()
        user_input = listen() if use_mic == "yes" else input(f"{user_name}: ").strip()

        if user_input.lower() in ["exit", "quit", "goodbye"]:
            speak("Goodbye! Take care.")
            break

        # Try system commands
        command_result = handle_command(user_input)
        if command_result:
            speak(command_result)
            continue

        # Generate response
        response = fallback_response(user_input, user_name)
        speak(response)


if __name__ == "__main__":
    main()
