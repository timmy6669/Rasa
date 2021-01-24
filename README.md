# Rasa
Chatbot chat with Mycroft

def initialize(self):
    # Set the address of your Rasa's REST endpoint
    self.conversation_active = False
    self.convoID = 1
    self.RASA_API = "/webhooks/rest/webhook"
    self.messages = []

def query_rasa(self, prompt=None):
    if self.conversation_active == False:
        return
    if prompt is None and len(self.messages) > 0:
        prompt = self.messages[-1]
    # Speak message to user and save the response
    msg = self.get_response(prompt, num_retries=0)
    # If user doesn't respond, quietly stop, allowing user to resume later
    if msg is None:
        return
    # Else reset messages
    self.messages = []
    # Send post requests to said endpoint using the below format.
    # "sender" is used to keep track of dialog streams for different users
    data = requests.post(
        self.RASA_API, json = {"message" : msg, "sender" : "user{}".format(self.convoID)}
    )
    # A JSON Array Object is returned: each element has a user field along
    # with a text, image, or other resource field signifying the output
    # print(json.dumps(data.json(), indent=2))
    for next_response in data.json():
        if "text" in next_response:
            self.messages.append(next_response["text"])
    # Output all but one of the Rasa dialogs
    if len(self.messages) > 1:
        for rasa_message in self.messages[:-1]:
            print(rasa_message)
 
    # Kills code when Rasa stop responding
    if len(self.messages) == 0:
        self.messages = ["no response from rasa"]
        return
    # Use the last dialog from Rasa to prompt for next input from the user
    prompt = self.messages[-1]
    # Allows a stream of user inputs by re-calling query_rasa recursively
    # It will only stop when either user or Rasa stops providing data
    return self.query_rasa(prompt)

@intent_handler(IntentBuilder("StartChat").require("Geronimo"))
def handle_talk_to_rasa_intent(self, message):
    self.convoID += 1
    self.conversation_active = True
    prompt = "start"
    self.query_rasa(prompt)
    
# Resume chat activator that would resume the conversation thread of a chat
@intent_handler(IntentBuilder("ResumeChat").require("Resume"))
def handle_resume_chat(self, message):
    self.conversation_active = True
    self.query_rasa()

def stop(self):
    self.conversation_active = False
