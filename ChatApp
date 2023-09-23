import React, { useState, useEffect, useRef } from 'react';
import { SpeechConfig, AudioConfig, SpeechRecognizer } from 'microsoft-cognitiveservices-speech-sdk';
import { OpenAI } from 'langchain/llms/openai';
import { BufferMemory } from 'langchain/memory';
import { ConversationChain } from 'langchain/chains';
import { SystemMessage } from 'langchain/schema';
import { ChatMessageHistory } from 'langchain/memory';
import './ChatApp.css';
import axios from 'axios';
import { auth, db, doc, getDocs, where, collection, query, addDoc } from './firebase';
import * as chrono from 'chrono-node';
import { useNavigate } from 'react-router-dom';

async function textToSpeech(text, voice_id) {
    {omitted}
    
    const response = await axios.post(`{omitted}`, {
      text: text,
      model_id: "eleven_monolingual_v1",
      voice_settings: {
        "stability": 0,
        "similarity_boost": 0,
        "style": 0.5,
        "use_speaker_boost": true,
        "optimize_streaming_latency": 4
      }
    }, {
      headers: {
        'xi-api-key': API_KEY
      },
      responseType: 'arraybuffer'
    });
    
    // Create a new AudioContext
    const audioContext = new (window.AudioContext || window.webkitAudioContext)();
    
    // Decode the audio data from the response
    audioContext.decodeAudioData(response.data, function(buffer) {
      // Create a new buffer source node
      const source = audioContext.createBufferSource();
      source.buffer = buffer;
      
      // Connect the source node to the audio context's destination (the speakers)
      source.connect(audioContext.destination);
      
      // Start playing the audio
      source.start(0);
    }, function(e) {
      console.error("Error decoding audio data", e);
    });
}

function ChatApp() {
    const [messages, setMessages] = useState([]);
    const [newMessage, setNewMessage] = useState('');
    const [company, setCompany] = useState('');
    const [position, setPosition] = useState('');
    const [intention, setIntention] = useState('');
    const [isSetupComplete, setIsSetupComplete] = useState(false);
    const [isListening, setIsListening] = useState(false);
    const recognizerRef = useRef(null);
    axios.defaults.baseURL = '';
    const [selectedSlot, setSelectedSlot] = useState(null); 

    const speechConfig = SpeechConfig.fromSubscription({omitted});
    const audioConfig = AudioConfig.fromDefaultMicrophoneInput();
    recognizerRef.current = new SpeechRecognizer(speechConfig, audioConfig);
    const [llm] = useState(new OpenAI({ openAIApiKey: process.env.REACT_APP_OPENAI_API_KEY }));
    const [memory, setMemory] = useState(null);
    const [chain, setChain] = useState(null);
    const [userMessages, setUserMessages] = useState([]);
    const navigate = useNavigate();

// Helper function to create all possible slots for a given day
const createAllPossibleSlots = (givenDate) => {
    const allSlots = [];
    const startHour = 9; // 9 AM
    const endHour = 17; // 5 PM
    const slotDuration = 60; // 60 minutes
  
    for (let hour = startHour; hour < endHour; hour++) {
      const start = new Date(givenDate);
      start.setHours(hour);
      start.setMinutes(0);
      start.setSeconds(0);
  
      const end = new Date(start);
      end.setMinutes(end.getMinutes() + slotDuration);
  
      allSlots.push({ start, end });
    }
  
    return allSlots;
  };
  
  // Helper function to check if two slots overlap
  const overlap = (slot, bookedSlot) => {
    const slotStart = new Date(slot);
    const slotEnd = new Date(slotStart);
    slotEnd.setHours(slotEnd.getHours() + 1); // Assuming 1-hour slots
    const bookedStart = new Date(bookedSlot.startTime);
    const bookedEnd = new Date(bookedSlot.endTime);
    return (bookedStart >= slotStart && bookedStart < slotEnd) || (bookedEnd > slotStart && bookedEnd <= slotEnd);
  };
  
  const fetchAvailableSlots = async (givenDate) => {
    console.log('Fetching available slots for:', givenDate);
    const allSlots = createAllPossibleSlots(givenDate);
  
    try {
      const slotsRef = collection(db,{omitted});
      const q = query(slotsRef, where('date', '==', givenDate));
      const querySnapshot = await getDocs(q);
  
      const bookedSlots = [];
      querySnapshot.forEach(doc => bookedSlots.push({ id: doc.id, ...doc.data() }));
  
      const availableSlots = allSlots.filter(slot => !bookedSlots.some(bookedSlot => overlap(slot.start, bookedSlot)));

      console.log('Available slots:', availableSlots);
      return availableSlots;
    } catch (error) {
      console.error('Error fetching available slots:', error);
      return [];
    }
  };

  const bookAppointment = async (selectedSlot) => {
    console.log('Booking appointment for slot:', selectedSlot);
    // Assuming the selected slot contains 'start' and 'end' properties
    let title = auth.currentUser?.uid || "Default title"; // Safe check

    const generateSummaryText = (userMessages) => {
        const chatMessages = userMessages.join(' '); // Joining the user messages into a single string
        return `From the following chat messages, detect the user's intention and only provide me 
        with a 6 word summary of the intention without any additional information. The chat messages: ${chatMessages}`;
    };
    const summary = generateSummaryText(userMessages);
    try {
        const openAIResponse = await processSummaryWithOpenAI(summary); // Send summary to OpenAI for processing
        if (openAIResponse && openAIResponse.response) {
            title = openAIResponse.response;
            console.log('Successfully received summary from OpenAI:', openAIResponse);
        } else {
            console.warn('Received empty summary or invalid structure from OpenAI');
        }
    } catch (error) {
        console.error('Error getting response from OpenAI:', error);
    }
    try {
        const appointmentRef = collection(db, {omitted});
        await addDoc(appointmentRef, {
            start: selectedSlot.start,
            end: selectedSlot.end,
            title: title // Assuming you have the user ID
        });    
        console.log('Appointment successfully booked.'); // Added log
        return true; // Indicate success
    } catch (error) {
        console.error('Error booking appointment:', error);
        return false; // Indicate failure
    }
};
const handleSlotSelection = async (selectedSlot) => {
        // Set the selected slot as the current selection
        const success = await bookAppointment(selectedSlot);
        setSelectedSlot(selectedSlot);
    
        console.log('Selected slot:', selectedSlot); // Log the selected slot
        // Trigger the appointment booking
        bookAppointment(selectedSlot).then(success => {
            if (success) {
                setMessages(prevMessages => [...prevMessages, { type: 'bot', content: `Your appointment has been successfully booked.` }]);
            } else {
                setMessages(prevMessages => [...prevMessages, { type: 'bot', content: `Sorry, there was an error booking your appointment. Please try again later.` }]);
            }
        });
    };  
      const handleSend = async () => {
        const result = await chain.call({ input: newMessage });
        console.log(result);
        if (result && result.response) {
            // Append the new user and bot messages to the message list
            setMessages(prevMessages => [...prevMessages, { type: 'user', content: newMessage }, { type: 'bot', content: result.response }]);
    
            // Call the textToSpeech function and pass the bot's response
            await textToSpeech(result.response);
    
            if (result.response.toLowerCase().includes("when would you like to book the appointment?") || newMessage.toLowerCase().includes("book an appointment")) {
                handleAppointmentRequest(newMessage);
            }
        }
        // Clear the input field
        setNewMessage('');
        setUserMessages(prevMessages => [...prevMessages, newMessage]);
    };
    const handleAppointmentRequest = async (userInput) => {
        console.log('User is trying to book an appointment.');
        const parsedResults = chrono.parse(userInput); // Parse the user's message for date and time
    
        if (parsedResults.length > 0) {
            const parsedDate = parsedResults[0].start.date(); // Extract the date
            const availableSlots = await fetchAvailableSlots(parsedDate);
    
            // Filter available slots based on the user's requested time, if specified
            const parsedTime = parsedResults[0].start;
            let selectedSlots = availableSlots;
            if (parsedTime.isCertain('hour') && parsedTime.isCertain('minute')) {
                const requestedHour = parsedTime.get('hour');
                const requestedMinute = parsedTime.get('minute');
                selectedSlots = availableSlots.filter(slot =>
                    slot.start.getHours() === requestedHour && slot.start.getMinutes() === requestedMinute
                );
            }
    
            // If exactly one slot matches, automatically select and book it
            if (selectedSlots.length === 1) {
                handleSlotSelection(selectedSlots[0]);
            } else {
                setSelectedSlot(selectedSlots); // Store all available slots
                setMessages(prevMessages => [...prevMessages, { type: 'bot', content: `Please select a slot from these options: ${selectedSlots.map(slot => `${slot.start.toLocaleTimeString()} - ${new Date(slot.start.getTime() + 60 * 60 * 1000).toLocaleTimeString()}`).join(', ')}` }]);
            }
        } else {
            setMessages(prevMessages => [...prevMessages, { type: 'bot', content: `I'm sorry, I couldn't understand the date and time you provided. Could you please specify them again?` }]);
        }
    };
    
    
    
    useEffect(() => {
        const speechConfig = SpeechConfig.fromSubscription({omitted});
        const audioConfig = AudioConfig.fromDefaultMicrophoneInput();
        recognizerRef.current = new SpeechRecognizer(speechConfig, audioConfig);
        recognizerRef.current.recognized = async (s, e) => {
            if (e.result.text && e.result.text.trim() !== '' && e.result.text.toLowerCase().includes("book an appointment")) {
                handleAppointmentRequest(e.result.text);
            }
            if (e.result.text && e.result.text.trim() !== '') {
                console.log(`RECOGNIZED: Text=${e.result.text}`);
                setNewMessage(e.result.text);
                if (e.result.text.length > 5) {
                    // Run the chain with user's input and get the result
                    const result = await chain.call({ input: e.result.text });
        
                    console.log(result); // log the result to see what's being returned

                    setUserMessages(prevMessages => [...prevMessages, e.result.text]);

        
                    if(result && result.response) {
                        // Append the new user and bot messages to the message list
                        setMessages(prevMessages => [...prevMessages, { type: 'user', content: e.result.text }, { type: 'bot', content: result.response }]);
                        // Call the textToSpeech function and pass the bot's response
                        const audioData = await textToSpeech(result.response);
                        // Now you can use this audio data to play the audio
                    } else {
                        console.error("Invalid result:", result);
                    }
                } else {
                    console.log("User input is too short or undefined. Please provide a longer input.");
                }
            } else {
                console.log("No valid user input detected. Ignoring background noise or silence.");
            }
            // Start listening again
            setIsListening(true);
            recognizerRef.current.startContinuousRecognitionAsync();
        };
        return () => {
            recognizerRef.current.close();
        };
    }, [chain]); // Removed 'messages' from the dependency array
    
    const toggleListening = () => {
        if (isListening) {
            recognizerRef.current.stopContinuousRecognitionAsync(
                () => {
                    console.log("Stopped continuous recognition");
                },
                (error) => {
                    console.error("Error stopping continuous recognition: ", error);
                }
            );
            setIsListening(false);
        } else {
            recognizerRef.current.startContinuousRecognitionAsync(
                () => {
                    console.log("Started continuous recognition");
                },
                (error) => {
                    console.error("Error starting continuous recognition: ", error);
                }
            );
            setIsListening(true);
        }
    };
    

    useEffect(() => {
        const firstMessage = "Hi! How are you?";
        setMessages([{ type: 'bot', content: firstMessage }]);
    }, []);

    const handleSetup = () => {
        let companyInfo = `You work at : ${company}\nYour position is: ${position}\nYour Intention is: ${intention} and you maximally use 100 letters. You ask questions to understand the customers needs and you try to book appointments \n`;
        const initialMessage = new SystemMessage(companyInfo);

        const chatHistory = new ChatMessageHistory([initialMessage]);

        let mem = new BufferMemory({
            chatHistory,

        });

        let ch = new ConversationChain({ llm, memory: mem });

        // Update the state with the newly created memory and chain
        setMemory(mem);
        setChain(ch);

        setIsSetupComplete(true);
        setNewMessage('');

    }


    async function processSummaryWithOpenAI(summary) {
        console.log("Sending summary request to OpenAI...");
        try {
            const res = await chain.call({ input: summary }); 
            console.log({ res });
            return res;
        } catch (error) {
            console.error('Error processing summary with OpenAI:', error);
            return null;
        }
    }
    

    return (     
        <div>
            {!isSetupComplete ? (
                <>
<div className="setup-header"> {/* Outer div retains original styling */}
  <div className="header-container"> {/* New container for the header */}
    <h1> Welcome!</h1>
  </div>
</div>

                    <input
                        type="text"
                        placeholder="Company"
                        onChange={e => setCompany(e.target.value)}
                    />
                    <input
                        type="text"
                        placeholder="Position"
                        onChange={e => setPosition(e.target.value)}
                    />
                    <input
                        type="text"
                        placeholder="Intention"
                        onChange={e => setIntention(e.target.value)}
                    />
                    <button onClick={handleSetup}>Complete Setup</button>
                </>
            ) : (

                <>
                <button onClick={() => navigate('/calendar')}>Go back to Calendar</button>

                {selectedSlot && selectedSlot.length > 0 && (
                    <div>
                        <div>Please select a slot from these options:</div>
                        {selectedSlot.map((slot, index) => (
                            <button key={index} onClick={() => handleSlotSelection(slot)}>
                                {slot.start.toLocaleTimeString()} - {new Date(slot.start.getTime() + 60 * 60 * 1000).toLocaleTimeString()}
                            </button>
                        ))}
                    </div>
                )}
                {messages.map((message, index) => (
                    <div key={index} className={message.type}>
                        {message.content}
                    </div>
                ))}
                <div className="input-area">
                    <button onClick={toggleListening}>
                        {isListening ? 'Stop Listening' : 'Start Listening'}
                    </button>
                    <input
                        type="text"
                        value={newMessage || ''}
                        onChange={e => setNewMessage(e.target.value)}
                    />
                    <button onClick={handleSend}>Send</button>
                </div>
            </>
        )}
    </div>
);
}
export default ChatApp; 
