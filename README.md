import tkinter as tk
import pyttsx3
import speech_recognition as sr
import random
import datetime
import requests
import wikipedia 
import threading 
import time 
import sys 

# Import the Google GenAI library for the AI Brain
from google import genai
from google.genai.errors import APIError as GeminiAPIError


#  CHITTI CONFIGURATION & SETUP


# 1. PASTE YOUR VALID GEMINI API KEY HERE 
GEMINI_API_KEY = "AIzaSyByUm3qdVNY39bGIVejgyBP3AawtLQi7iM" 
# 2. Optional: For Weather function (Get from OpenWeatherMap)
OPENWEATHER_API_KEY = "your_openweather_api_key" 

#  VOICE ENGINE SETUP
engine = pyttsx3.init()
engine.setProperty('rate', 150)
engine.setProperty('volume', 1.0)

#  GLOBAL TEXT TRACKER
LAST_CHITTI_TEXT = "" # Global variable to store CHITTI's last speech

#  Voice Setup 
try:
    voices = engine.getProperty('voices')
    VOICE_INDEX = 0 
    for i, voice in enumerate(voices):
        # Prioritize finding a female voice (Zira/Zara) or a common male voice (David)
        if "zira" in voice.name.lower() or "zara" in voice.name.lower():
            VOICE_INDEX = i
            break
        elif "david" in voice.name.lower():
            VOICE_INDEX = i
            break
    if len(voices) > VOICE_INDEX:
        engine.setProperty('voice', voices[VOICE_INDEX].id) 
except Exception as e:
    print(f"Warning: Error setting up voice engine: {e}")
    
#  JOKE SYSTEM GLOBALS 
joke_queue = [] 

#  KEYWORD CATEGORIES 
KEYWORDS = {
    "joke": ["joke", "funny", "hasna", "laugh"],
    "time": ["time", "samay", "clock", "kitne baje"],
    "weather": ["weather", "mausam", "barish", "sunny", "cloudy"],
    "clear chat": ["clear chat", "delete chat", "reset chat", "clear history"],
    "search": ["what is", "tell me about", "who is", "explain", "meaning of", "research on"],
    "help": ["help", "commands", "madad", "kya karu"],
}

#  CUSTOM HUMOR & FILLERS
FUNNY_FILLERS = [
    "Arey boss, sunn! Yeh toh wahi baat ho gayi ki...", "Ekdum faadu information hai yeh! Toh, baat aise toh hai, boss! ki...",
]
HUMOR_MODIFIERS = [
    ("is", "toh hai, boss!"), ("was", "tha, par abhi kya hai?"),
    (".", ". (Aur yeh toh sabko pata hona chahiye!)"),
]

#  EXPANDED JOKE LIST (MASTER COPY) 
JOKE_LIST = [
    "Teacher: Tum itne din kyu nahi aaye? Student: Sir, ghar mein wifi nahi tha! 😆",
    "Pappu: Mummy mujhe chocolate chahiye! Mummy: Thappad chahiye kya? Pappu: Dono de de 😅",
    "Robot: Mujhe bhi pyaar ho gaya... par battery low thi 💔",
    "Why was the math book sad? Because it had too many problems! 🤓",
    "Waiter: Do you want your coffee black? Me: What other colours do you have? 🌈",
    "Husband to wife: Tumhe pata hai, jab tum ghusse mein hoti ho na... tum aur bhi sundar lagti ho! Wife: (Smiling) Sach mein? Husband: Haan, phir toh tum theek lagti ho. 😏",
    "Aaj kal toh bas ek hi tension hai, padhai ya PUBG? Dil bolta hai PUBG, dimaag bolta hai... abhi bhi PUBG. 🎮",
    "Arey, tumhari shirt par itna bada daag kaisa? Dost: Kya karun yaar, main Dahi Bhalla kha raha tha. Me: Toh daag kaisa? Dost: Bhalla toh theek tha, par dahi lad raha tha! 😂",
    "Doctor: Aapke teen daant kaise toot gaye? Patient: Maine roti khayi thi. Doctor: Par roti se teen daant kaise? Patient: Woh actually biscuit thi! 🍪",
    "Dost 1: Yaar, meri wife ne mujhe 'darwaja' bola. Dost 2: Kyun? Dost 1: Kehti hai, din bhar khula rehta hai aur raat ko band karna padta hai! 🚪",
    "A joke about a programmer who died in the shower. He left a comment: 'Too much washing. Little compiling.'",
    "Why did the invisible man turn down the job offer? He couldn't see himself doing it!",
] 



# CHITTI CORE FUNCTIONS (GEMINI IMPLEMENTATI
 
def _speak_in_thread(text):
    """Helper function to run TTS without blocking the GUI."""
    try:
        time.sleep(0.05) 
        engine.stop() 
        engine.say(text)
        engine.runAndWait()
        
    except Exception as e:
        print(f"TTS Engine Error: Could not speak the text '{text}'. Error: {e}")

def chitti_speak(text):
    """Inserts text into the GUI and starts TTS in a separate thread.
        Also updates the LAST_CHITTI_TEXT variable."""
    global LAST_CHITTI_TEXT
    
    # Update GUI immediately 
    chat_history.insert(tk.END, f"CHITTI: {text}\n")
    chat_history.see(tk.END) 
    
    # Store the text spoken (for the Read Text button)
    LAST_CHITTI_TEXT = text
    
    # Start speaking in a new thread to prevent GUI freeze
    threading.Thread(target=_speak_in_thread, args=(text,)).start()

def speak_user_text():
    """Reads the text currently in the input box or the last CHITTI message."""
    global LAST_CHITTI_TEXT
    text_to_read = entry.get().strip()
    
    if text_to_read:
        # If user typed something new, read that
        entry.delete(0, tk.END) 
        chat_history.insert(tk.END, f"READING (User Text): {text_to_read}\n")
        chitti_speak(text_to_read)
    elif LAST_CHITTI_TEXT:
        # If input box is empty, read the last thing CHITTI said
        chat_history.insert(tk.END, f"READING (Last CHITTI Msg): {LAST_CHITTI_TEXT}\n")
        # Call the speak function directly without updating LAST_CHITTI_TEXT again
        threading.Thread(target=_speak_in_thread, args=(LAST_CHITTI_TEXT,)).start()
    else:
        # If both are empty
        chitti_speak("Bhai, pehle kuch likh toh do, ya fir mere bolne ka wait karo! Hawa nahi padh sakta main.")
    
    chat_history.see(tk.END)


def listen_thread():
    """Starts the listening process in a separate thread to keep the GUI responsive."""
    threading.Thread(target=_listen_for_command).start()

# Fixed _listen_for_command function with improved sensitivity and device index
def _listen_for_command():
    """Captures user input via microphone."""
    recognizer = sr.Recognizer()
    
    # ADJUSTED FOR MICROPHONE FIX: Set a specific index (often 0 or 1 on systems)
    MIC_INDEX = 0 
    
    recognizer.dynamic_energy_threshold = False  
    recognizer.energy_threshold = 100        
    recognizer.pause_threshold = 1.0        

    try:
        # Use the specific device index found 
        with sr.Microphone(device_index=MIC_INDEX) as source:
            chitti_speak("Bol bhai, sun raha hoon...")
            
            recognizer.adjust_for_ambient_noise(source, duration=2.5) 
            
            try:
                # Increased timeout to 10 seconds 
                audio = recognizer.listen(source, timeout=10, phrase_time_limit=20) 
            except sr.WaitTimeoutError:
                chitti_speak("Time out ho gaya, jaldi bol na! 😴")
                return
                
            try:
                command = recognizer.recognize_google(audio, language="en-IN") 
                
                root.after(0, lambda: (entry.delete(0, tk.END), entry.insert(0, command)))
                
                chitti_speak(f"Maine suna: '{command}'. Confirm karne ke liye Enter daba, ya 'ASK CHITTI' button dabao.")
                
            except sr.UnknownValueError:
                chitti_speak("Sorry bhai, sun nahi paya 😅")
            except sr.RequestError:
                chitti_speak("Internet ka issue hai lagta hai. Google bola 'no service'! 😭")
            except ValueError:
                chitti_speak(f"Microphone Index {MIC_INDEX} galat hai, ya device disconnect ho gaya. Check karo! 🛑")

    except Exception as e:
        if "No module named 'pyaudio'" in str(e):
            chitti_speak("PyAudio missing! 'pip install pyaudio' karke try kar. Microphone general error: {e}")
        else:
            chitti_speak(f"Microphone general error: {e}")


# LLM Response Function (USES GEMINI)
def get_llm_response(prompt):
    """Sends the user message to the Gemini API with a personality prompt, 
       with enhanced error handling for blocked/empty responses and retries."""
    global GEMINI_API_KEY
    
    #  CRITICAL API KEY DIAGNOSTIC CHECK 
    if GEMINI_API_KEY == "AIzaSyAqbcjhfvbkF_KMYbm68ZH5dWDpWgRaBIQ": 
        print("\n--- CRITICAL API KEY ALERT ---")
        print("ERROR: You are using the default placeholder API key.")
        return "Critical API Key Error! Bhai, asli waali Gemini key daal do. Placeholder key kaam nahi karti. 🔑"
    #  END CRITICAL API KEY DIAGNOSTIC CHECK 
    
    try:
        # Initial check for client setup failure
        client = genai.Client(api_key=GEMINI_API_KEY)
    except Exception:
        return "Critical Error: Gemini Client setup failed. Is the 'google-genai' library installed?"

    # SYSTEM PROMPT FOR FUNNY/SARCASTIC STYLE 
    system_prompt = (
        "You are a friendly, witty, and extremely sarcastic AI comedian named CHITTI 7.0 "
        "Ultimate. Your only goal toh hai, boss! to make the user laugh. Always reply with highly humorous, "
        "witty, and slightly sarcastic remarks. Inject jokes and emojis into every response. "
        "Keep the tone light and entertaining."
    )
    
    MAX_RETRIES = 3
    # Start retry loop
    for attempt in range(MAX_RETRIES):
        try:
            # Generate the content using the faster Gemini Flash model
            response = client.models.generate_content(
                model='gemini-2.5-flash',
                contents=[
                    {"role": "user", "parts": [{"text": f"{system_prompt}\n\nUser: {prompt}"}]}
                ],
                config={
                    'max_output_tokens': 150,
                    'temperature': 0.8, 
                }
            )
            
            # 1. Primary Success Check: Return if text exists and is not just empty whitespace
            if response.text and response.text.strip():
                return response.text
            
            # 2. Secondary Failure Check (Safety/Block)
            if response.candidates and response.candidates[0].finish_reason:
                finish_reason = response.candidates[0].finish_reason

                if 'SAFETY' in finish_reason:
                    return "🚨 Arey bhai! Mera AI-brain bola 'Safety first!'. Shayad kuch zyada hi spicy bol raha tha main. Try a safer topic! 🌶"
                
                # If the response is empty but the reason is not safety, and it's the last attempt, report error.
                if attempt == MAX_RETRIES - 1:
                    return "🤔 Lagta hai mera brain Hang ho gaya tha! Koi message nahi aaya, ya bolte-bolte ruk gaya. Main phir se try karta hoon, tu tension mat le! 🔄"

            # If no text and no explicit failure reason, or if the finish_reason implies an incomplete response, we retry.
            
        except GeminiAPIError as e:
            # Check for invalid key explicitly, which should not be retried.
            if "API_KEY_INVALID" in str(e):
                return "Gemini API key galat hai! Check karle bhai. 🔑"
            
            # If it's the final attempt, report the network/server error.
            if attempt == MAX_RETRIES - 1:
                return f"Gemini API se baat nahi ho pa rahi. Server error: {e}. Thoda ruk ja, try again. 🌐"
            
        except Exception as e:
            # Catch unexpected errors like network issues or library problems.
            if attempt == MAX_RETRIES - 1:
                return f"Kuch toh gadbad hai daya! Unknown error: {e}"

        # Exponential Backoff (wait before next attempt): 1s, 2s, 4s...
        sleep_time = 2 ** attempt
        time.sleep(sleep_time)
        
    # Final fallback if all retries failed (should not be reached)
    return "🤷‍♂ Bhai, lagta hai AI-server ne aaj strike kar di hai. Mujhe koi message nahi mila. Thoda wait kar ya phir se puchh! ⏳"

def get_non_repeating_joke():
    """Shuffles the entire list when exhausted, ensuring full cycle without repetition."""
    global joke_queue
    
    if not joke_queue:
        joke_queue = list(JOKE_LIST) 
        random.shuffle(joke_queue)          
        
    return joke_queue.pop(0)

def get_time():
    """Returns the current local time."""
    now = datetime.datetime.now()
    return f"Abhi baj rahe hain {now.strftime('%I:%M %p')} 🕒"

def get_weather(city="Chhatrapati Sambhajinagar"):
    """Fetches and formats local weather information."""
    if OPENWEATHER_API_KEY == "" or OPENWEATHER_API_KEY == "your_openweather_api_key":
        return "Pehle *OpenWeather API Key* daal de bhai! Warna weather nahi bataunga. 🔑"
    
    url = f"http://api.openweathermap.org/data/2.5/weather?q={city}&appid={OPENWEATHER_API_KEY}&units=metric"
    
    try:
        response = requests.get(url, timeout=5).json()
        if response.get("cod") == 200:
            temp = response['main']['temp']
            desc = response['weather'][0]['description']
            return f"{city} mein {desc} hai aur temperature hai {temp}°C 🌦"
        else:
            return "City ka naam nahi mila ya API key galat hai. Check karle bhai! 😅"
    except Exception:
        return "Network ka locha hai lagta hai. Ya phir yeh information 'top secret' hai! 🤫"

def get_detailed_answer(query):
    """Searches Wikipedia and applies the funny filter."""
    if not query:
        return "Pehle bata toh de kya search karna hai? Khali haath kya karoon main?😜"
        
    try:
        summary = wikipedia.summary(query, sentences=3, auto_suggest=False, redirect=True)
        
        # Apply funny filter
        funny_reply = random.choice(FUNNY_FILLERS) + "\n\n"
        factual_part = summary
        for original, funny in HUMOR_MODIFIERS:
            # We only apply the modifier if the original word is found
            factual_part = factual_part.replace(" " + original + " ", " " + funny + " ")
            
        funny_reply += factual_part
        funny_reply += "\n\nAisa hai yeh 'item'! Samajh gaya na, ya aur 'Mirchi' daalun? 😂"
        
        return funny_reply

    except wikipedia.exceptions.PageError:
        return f"Arey, '{query}' naam ka koi 'page' toh mila hi nahi. Galat jagah search kar raha hai kya? 🔍"
    except wikipedia.exceptions.DisambiguationError as e:
        return get_detailed_answer(e.options[0])
    except Exception:
        return "Network ka locha hai lagta hai. Ya phir yeh information 'top secret' hai! 🤫"

def get_help_message():
    """Provides a list of available commands."""
    help_text = (
        "Main yeh sab kar sakta hoon, boss! 💪\n"
        "1. *🎤SPEAK:* Mic on karke bol. Type karne ke baad Enter/Button dabaana mat bhoolna!\n"
        "2. *😂 JOKE:* Non-repeating jokes ke liye! \n"
        "3. *❓ WHAT IS...:* Wikipedia search karta hoon (e.g., 'what is Jupiter').\n"
        "4. *🕒 TIME:* Time bataunga.\n"
        "5. *Mausam (Weather):* Bas 'weather' bol de ya 'mausam' to know the status. \n"
        "6. *📣 READ TEXT:* *NEW!* Reads the text in the box OR the last thing I said.\n"
        "7. *AI Chat (Gemini Brain!):* Agar koi command match nahi hua, toh main *AI se funny style mein baat* karta hoon. "
    )
    return help_text

# Respond (The main handler)
def respond():
    """Processes user input and generates the appropriate response."""
    user_input = entry.get().lower().strip()
    entry.delete(0, tk.END) # Clear the box immediately
    
    if not user_input:
        return 
        
    chat_history.insert(tk.END, f"You: {user_input}\n")
    chat_history.see(tk.END)

    if "bye" in user_input:
        chitti_speak("Chal theek hai bhai, milte next update mein! 💥👋")
        root.quit()
        return

    is_special_command = False
    final_reply = ""
    
    # 1. Check for command categories
    if any(w in user_input for w in KEYWORDS["clear chat"]):
        chat_history.delete(1.0, tk.END)
        final_reply = "Chat history saaf kar diya bhai 🧹"
        is_special_command = True
    elif any(w in user_input for w in KEYWORDS["time"]):
        final_reply = get_time()
        is_special_command = True
    elif any(w in user_input for w in KEYWORDS["weather"]):
        final_reply = get_weather()
        is_special_command = True
    elif any(w in user_input for w in KEYWORDS["joke"]):
        final_reply = get_non_repeating_joke()
        is_special_command = True
    elif any(w in user_input for w in KEYWORDS["help"]):
        final_reply = get_help_message()
        is_special_command = True
    elif any(w in user_input for w in KEYWORDS["search"]):
        search_word = next((w for w in KEYWORDS["search"] if w in user_input), "")
        query = user_input.replace(search_word, "", 1).strip()
        final_reply = get_detailed_answer(query)
        is_special_command = True
    
    # 2. If no command matched, send to the LLM (Gemini)
    if not is_special_command:
        # This calls the Gemini API
        final_reply = get_llm_response(user_input)

    # 3. CRITICAL SAFETY CHECK: If the LLM function returns None (due to unhandled error)
    if final_reply is None:
        final_reply = "⚡ Arey, mera AI brain bilkul khali ho gaya! *Yeh toh serious locha hai. Iska matlab 99% yeh hai ki **Gemini API Key galat hai, ya network block ho gaya hai.* Check your key and connection!"

    chitti_speak(final_reply)



# 🖼 GUI SETUP (NO CHANGES MADE HERE)

def on_closing():
    """Handle proper cleanup before closing the GUI."""
    # We rely on runAndWait() completing in its thread. We just destroy the GUI.
    root.destroy()
    sys.exit(0)

root = tk.Tk()
root.title("CHITTI 7.0 Ultimate 🤖 - Funny AI (Gemini Enabled)")
root.geometry("650x750") 
root.configure(bg="#73ADD1") 
root.protocol("WM_DELETE_WINDOW", on_closing) # Handle window closing event

# Title Frame 
title_frame = tk.Frame(root, bg="#73ADD1")
title_frame.pack(pady=(20, 10), fill=tk.X)

# Main Title Label
title = tk.Label(title_frame, text="🔥 C H I T T I   7 . 0   U L T I M A T E 🔥", 
                 font=("Times new Roman", 24, "bold"), 
                 bg="#FFFFFF", 
                 fg="#FF0000") 
title.pack()

# Chat History Window 
chat_history = tk.Text(root, 
                        bg="#FFFFFF", 
                        fg="#1F1F1F", 
                        font=("Courier New", 16), 
                        height=18, 
                        wrap="word", 
                        borderwidth=0, 
                        relief="solid", 
                        highlightthickness=2, 
                        highlightbackground="#FF0000") 
chat_history.pack(pady=10, padx=20, fill=tk.BOTH, expand=True)

# --- Input Entry ---
entry = tk.Entry(root, 
                  font=("Arial", 14), 
                  bg="#333333", 
                  fg="white", 
                  insertbackground="#6AC759", 
                  bd=2, 
                  relief=tk.FLAT)
entry.pack(pady=10, fill=tk.X, padx=20)

# --- Button Frame (Centralized) ---
btn_frame = tk.Frame(root, bg="#FBFB95")
btn_frame.pack(pady=(10, 20))

# Helper function for button command
def insert_and_respond(text):
    entry.delete(0, tk.END)
    entry.insert(0, text)
    respond()
    
# Button Styling 
btn_style = {"width": 12, 
              "activebackground": "#555555", 
              "activeforeground": "#39FF14", 
              "font": ("Arial", 10, "bold"), 
              "bd": 0,
              "relief": tk.FLAT,
              "cursor": "hand2"} 

# Defining specific colors
DEFAULT_BG = "#444444"
DEFAULT_FG = "white"
ASK_BG = "#39FF14" 
ASK_FG = "#0E0D0D" 
BYE_BG = "#FF5733" 
READ_BG = "#FFD700" # Gold for Read Text
READ_FG = "#0A0A0A" # Black for Read Text

# Buttons 
# ROW 0
tk.Button(btn_frame, text="🎤 SPEAK", command=listen_thread, **btn_style, bg=DEFAULT_BG, fg=DEFAULT_FG).grid(row=0, column=0, padx=8, pady=5)
tk.Button(btn_frame, text="😂 JOKE", command=lambda: insert_and_respond("tell me a joke"), **btn_style, bg=DEFAULT_BG, fg=DEFAULT_FG).grid(row=0, column=1, padx=8, pady=5)
tk.Button(btn_frame, text="❓ WHAT IS...", command=lambda: entry.insert(0, "what is "), **btn_style, bg=DEFAULT_BG, fg=DEFAULT_FG).grid(row=0, column=2, padx=8, pady=5)
tk.Button(btn_frame, text="🕒 TIME", command=lambda: insert_and_respond("what time is it"), **btn_style, bg=DEFAULT_BG, fg=DEFAULT_FG).grid(row=0, column=3, padx=8, pady=5)

# ROW 1 (Functional buttons)
tk.Button(btn_frame, text="🤖 ASK CHITTI", command=respond, **btn_style, bg=ASK_BG, fg=ASK_FG).grid(row=1, column=0, columnspan=2, padx=8, pady=10, sticky="ew") 
tk.Button(btn_frame, text="🧹 CLEAR", command=lambda: insert_and_respond("clear chat"), **btn_style, bg=DEFAULT_BG, fg=DEFAULT_FG).grid(row=1, column=2, padx=8, pady=10)
tk.Button(btn_frame, text="👋 BYE", command=lambda: insert_and_respond("bye"), **btn_style, bg=BYE_BG, fg="white").grid(row=1, column=3, padx=8, pady=10)

# ROW 2 (New Read Text Button)
tk.Button(btn_frame, text="📣 READ TEXT", command=speak_user_text, **btn_style, bg=READ_BG, fg=READ_FG).grid(row=2, column=0, columnspan=4, padx=8, pady=5, sticky="ew")

# Bind the Enter key to the respond function
root.bind('<Return>', lambda event=None: respond())

# Welcome message
chitti_speak("🤖 Namaste! Main hoon CHITTI 7.0 Ultimate, now with a *Funny Brain*! Ask me anything, full comedy style!")

# Launch the app
if _name_ == "_main_":
    root.mainloop()
