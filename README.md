# conversation_data
This repository contains the sample data used for training the **"Neural Conversational Model"** for the *Reminders* domain, in the context of [Haptik](http://www.haptik.ai)

## Overview
The corpus comprises of dyadic conversations from the Reminders domain. It is a collection of messages exchanged between Haptik users and chat assistants (responses from humans or the Rule Based ChatBot). The original corpus contains data collected over the past two years. This repository contains a sample data from the original corpus. The messages include casual queries, out of domain requests, push notifications sent to user to start a new conversation or to keep them engaged. The messages are observed to be mostly in English language but significant proportion of data also consists of code-mixed and code-switched messages - primarily English code-mixed with Hindi. In the original corpus, each message has rich metadata accompanying it – user info (age, gender, location, device, etc.), timestamps and type of message (e.g. simple text, UI element, form, etc). The UI elements are represented in the haptik specific custom format in the text corpus.

## Preprocessing
Before we train the Neural model, we perform several preprocessing steps, for efficient training. With preprocessing we try to normalize the data, reduce vocabulary size, convert raw text into actionables, remove unnecessary data.
The [sample_raw_unames.csv](sample_raw_unames.csv) file contains the data in raw format (without any processing). For maintaining privacy, we do not reveal the user-names and have replaced the names with *\_name\_* tag.  Following are the preprocessing steps followed:

1. **Removal of out of domain conversations**

Haptik being a personal assistant app, users tend to ask queries from other domains in Reminders service. The first step involves removal of conversations from the data which do not belong to Reminders domain. This step is performed using the in-house domain identifier. Even after this step, some conversations are not filtered out because of the domain shifts that happen midway through a conversation.

2. **Merge Reminder Notifications**

Users usually set multiple reminders on Haptik. For example, the drinking water reminder asks for how frequently should a user be reminded regarding the task. We often see a continuous series of a reminder notifications, in a conversation. In this step, we delete all such notifications except for the last one in the series. This reduces the number of outbound messages in data by a significant amount.

The [sample_pruned_chats_unames.csv](sample_pruned_chats_unames.csv) file contains the data after performing the steps 1,2. You will see a large difference in size of the data because of the reasons explained above.

3. **Replace Entities**

In this step we replace the original text of the named entities date, time ,phone number, user name and assistant name by the *“\_date\_”, “\_time\_”, “\_phone\_”, “\_name\_”, “\_name\_”* tags respectively. This reduces the size of vocabulary significantly. In our scenario, the value of the entity is not important but only its presence is important. A separate object maintains the entities identified (explained in hybrid). For identifying the entities, we use the in-house [NER](https://github.com/hellohaptik/chatbot_ner).

*Ex. Raw query:* Remind me to go to gym tomorrow at 5pm

*Preprocessed query:* Remind me to go to gym \_date\_ at \_time\_ 

4. **Structured messages**

This step involves handling of UI elements. In case of an UI element, we simply replace the text with the corresponding ID (every structured outbound UI element has an associated ID). In case of the filled UI elements sent by an user, we replace the element values with element keys for all the filled element fields.

*Ex. Query:* 
“Wake up call

Date: 22-10-2016

Time: 6:05 AM”

*Preprocessed query:* “wake up call date \_date\_ time \_time\_” 

5. **Extracting actions from message**

This is a crucial part of the preprocessing step. The corpus consists of raw text messages exchanged between user and the haptik assistant. But being an utility product, just a text response is not enough to cater the user queries. We need a mechanism to identify the action that needs to be performed. While setting up or cancelling a reminder, an acknowledgement message is sent to the user. In this step of preprocessing, we try to utilize these messages and tag them as an action. Whenever an action tag is predicted as a response by the neural model, we perform that action by calling corresponding APIs.

*Ex. Assistant Message:* Okay, done. We will remind you to take your medicine, via a call at 2:00 PM on Tue, 18 April. Take care :)

*Preprocessed message:* \_api\_call\_reminder\_medicine\_

*Action:* set a medicine reminder 

6. **Orthography**

In this step we first convert the string to lowercase. We remove all the punctuation marks. We replace all the numeric values with “\_numeral\_” tag. At the end of this step, the data can contain characters only from set (a-z) and a special character (“\_”). This step helps reduce the vocabulary size by a significant fraction. While this step helps training on smaller data, the downside of this approach is that, we have to introduce a post-processing step for making the response look appropriate.

The [sample_preprocessed.csv](sample_preprocessed.csv) file contains the data after performing steps 3, 4, 5 and 6.

## Training Data (Context - Response pairs)

For training a Neural Conversation Model, we need context - respose pair (to treat it as a SMT problem where the input and output are same semtence in two different languages).

For an example conversation like [U1 A1 U2 A2 A3 U3 A4] where Ui refers to a user message and Ai refers to an assistant message, Context-Response pairs are generated as follows:

Context [U1] Response [A1]

Context [U1 A1 U2] Response [A2]

Context [U1 A1 U2 A2] Response [A3]

Context [U1 A1 U2 A2 A3 U3] Response [A4]

- The file [sample_human_context_response.tsv](sample_human_context_response.tsv) contains such context response pairs, where the response is sent by a human assistant. While in the file [sample_bot_context_response.tsv](sample_bot_context_response.tsv), the responses are sent by bot.

- Different messages from the same speaker are separated by a particular token while messages representing end of turn i.e switching of speaker between user and agent are represented by a separate token.

- The context part is limited to maximum 160 words 

## Terminology

- **coll_id:** User ID

- **conv_no:** User's session number (one conversation -> one chat session)

- **Direction:** True -> Message is sent by user; False -> message is sent by Assistant (human or bot)

- **msg_type:** Various types of messages varying in formats (haptik specific). Some important msg types include *22* -> reminder notification; *0* -> sender is a human; *17* -> form
