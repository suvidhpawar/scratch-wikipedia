just put your username password and project id into this program


import scratchattach as sa
from scratchattach import Encoding
import wikipedia
from wikipedia.exceptions import DisambiguationError, PageError
import time
import warnings
from sense_hat import SenseHat
sense = SenseHat()
sense.clear()
warnings.filterwarnings('ignore', category=sa.LoginDataWarning)

USERNAME = enter username
PASSWORD = enter password
PROJECT_ID = enter project id
MAX_LEN = 256  # Scratch cloud variable limit

def get_wikipedia_summary(query):
    """Fetch a Wikipedia summary, auto-picking first disambiguation option if needed, then cut to fit limit."""
    try:
        try:
            # Try first without auto-suggest to reduce surprises
            summary = wikipedia.summary(query, sentences=3, auto_suggest=False)
        except DisambiguationError as e:
            if e.options:
                print(f"Disambiguation: trying first option '{e.options[0]}'")
                summary = wikipedia.summary(e.options[0], sentences=3, auto_suggest=False)
            else:
                return "Multiple meanings. Be more specific."
        except PageError:
            return "Wikipedia page not found."

        # Trim summary if encoded length exceeds cloud variable limit
        encoded_summary = Encoding.encode(summary)
        while len(str(encoded_summary)) > MAX_LEN and len(summary) > 5:
            summary = summary[:-1]  # remove last char
            encoded_summary = Encoding.encode(summary)

        if len(str(encoded_summary)) > MAX_LEN:
            summary = summary[:len(summary) - 3] + "..."
            encoded_summary = Encoding.encode(summary)

        return summary

    except Exception as e:
        return f"Error accessing Wikipedia: {str(e)[:100]}"

def main():
    session = sa.login(USERNAME, PASSWORD)
    cloud = session.connect_cloud(PROJECT_ID)
    events = cloud.events()

    @events.event
    def on_set(activity):
        if activity.var == "input":
            eninput = activity.value
            if eninput:
                decoded_input = Encoding.decode(eninput).strip()
                print(f"Decoded input: '{decoded_input}'")
                if decoded_input:
                    summary = get_wikipedia_summary(decoded_input)
                    encoded_summary = Encoding.encode(summary)

                    print(f"Final encoded length: {len(str(encoded_summary))} chars")
                    cloud.set_var("output", str(encoded_summary))
                else:
                    cloud.set_var("output", Encoding.encode("Please enter a valid search term."))
            else:
                cloud.set_var("output", Encoding.encode("Input is empty."))

    events.start()
    print("Wikipedia bot running. Press Ctrl+C to stop.")
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        events.stop()

if __name__ == "__main__":
    main()
