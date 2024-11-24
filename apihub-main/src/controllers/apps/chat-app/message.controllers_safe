import mongoose from "mongoose";
import axios from "axios";
import { ChatEventEnum } from "../../../constants.js";
import { Chat } from "../../../models/apps/chat-app/chat.models.js";
import { ChatMessage } from "../../../models/apps/chat-app/message.models.js";
import { emitSocketEvent } from "../../../socket/index.js";
import { ApiError } from "../../../utils/ApiError.js";
import { ApiResponse } from "../../../utils/ApiResponse.js";
import { asyncHandler } from "../../../utils/asyncHandler.js";
import { getLocalPath, getStaticFilePath, removeLocalFile } from "../../../utils/helpers.js";

// Common aggregation for fetching sender data along with message details
const chatMessageCommonAggregation = () => [
  {
    $lookup: {
      from: "users", // Lookup the "users" collection to get sender details
      foreignField: "_id", // Match sender's id
      localField: "sender", // with the "sender" field in the message collection
      as: "sender", // The result will be stored in the "sender" field of the message
      pipeline: [{ $project: { username: 1, avatar: 1, email: 1 } }], // Fetch only these fields from the user
    },
  },
  { $addFields: { sender: { $first: "$sender" } } }, // Flatten the array and use the first (and only) element
];

// Helper function to handle retries for Hugging Face API
const fetchWithRetry = async (url, data, headers, retries = 3, delay = 1000) => {
  let attempt = 0;
  while (attempt < retries) {
    try {
      const response = await axios.post(url, data, { headers });
      return response;
    } catch (error) {
      if (error.response?.status === 429) {
        console.warn(`Rate limit hit, retrying after ${delay / 1000} seconds...`);
        await new Promise((resolve) => setTimeout(resolve, delay)); // Delay before retry
        delay *= 2; // Exponential backoff
      } else {
        throw error; // Re-throw if the error is not a 429
      }
    }
    attempt++;
  }
  throw new ApiError(429, "Rate limit exceeded. Please try again later.");
};

// Fetch all messages for a specific chat
const getAllMessages = asyncHandler(async (req, res) => {
  const { chatId } = req.params;
  const selectedChat = await Chat.findById(chatId);

  if (!selectedChat) throw new ApiError(404, "Chat does not exist");
  if (!selectedChat.participants.includes(req.user?._id)) {
    throw new ApiError(400, "User is not a part of this chat");
  }

  const messages = await ChatMessage.aggregate([
    { $match: { chat: new mongoose.Types.ObjectId(chatId) } },
    ...chatMessageCommonAggregation(),
    { $sort: { createdAt: -1 } },
  ]);

  return res
    .status(200)
    .json(new ApiResponse(200, messages || [], "Messages fetched successfully"));
});

// Helper function to filter content using Hugging Face's API
async function filterContentWithHuggingFace(text) {
  const apiKey = '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'; // Replace with your Hugging Face API Key
  const url = 'https://api-inference.huggingface.co/models/unitary/toxic-bert'; // Toxicity detection model

  const headers = {
    'Authorization': `Bearer ${apiKey}`,
    'Content-Type': 'application/json',
  };

  const body = { inputs: text };

  try {
    const response = await fetchWithRetry(url, body, headers);
    return response.data;
  } catch (error) {
    throw new ApiError(500, "Error filtering content with Hugging Face API: " + error.message);
  }
}

// Send message and filter with Hugging Face API
const sendMessage = asyncHandler(async (req, res) => {
  const { chatId } = req.params;
  let { content } = req.body;

  if (!content && !req.files?.attachments?.length) {
    throw new ApiError(400, "Message content or attachment is required");
  }

  let moderationResponse;

  try {
    // Use Hugging Face moderation API for content filtering
    moderationResponse = await filterContentWithHuggingFace(content);

    // Log the moderation response for debugging
    console.log("Hugging Face moderation response:", moderationResponse);

    // Flatten the response since it's an array of arrays
    const labels = moderationResponse[0];  // Access the first array in the response

    // Define the toxic labels we want to check
    const toxicLabels = [
      'toxic', 
      'obscene', 
      'insult', 
      'severe_toxic', 
      'identity_hate', 
      'threat'
    ];

    // Check if any of the labels have a score greater than 0.5
    const isProfane = labels.some((label) => {
      // Check if the label is in the toxic labels list and the score is greater than 0.5
      return toxicLabels.includes(label.label) && label.score > 0.5;
    });

    if (isProfane) {
      // If message is profane, replace the content with a filtered message
      content = "Message contains inappropriate or offensive content";
    }
  } catch (error) {
    console.error("Error filtering content with Hugging Face API:", error);
    throw new ApiError(500, "Error filtering content");
  }

  const selectedChat = await Chat.findById(chatId);
  if (!selectedChat) throw new ApiError(404, "Chat does not exist");

  const messageFiles = [];
  if (req.files?.attachments?.length > 0) {
    req.files.attachments.forEach((attachment) => {
      messageFiles.push({
        url: getStaticFilePath(req, attachment.filename),
        localPath: getLocalPath(attachment.filename),
      });
    });
  }

  // Create the message in the database (whether it's the original or filtered message)
  const message = await ChatMessage.create({
    sender: new mongoose.Types.ObjectId(req.user._id),
    content,
    chat: new mongoose.Types.ObjectId(chatId),
    attachments: messageFiles,
  });

  // Update the chat's last message
  await Chat.findByIdAndUpdate(chatId, { lastMessage: message._id }, { new: true });

  const messages = await ChatMessage.aggregate([
    { $match: { _id: new mongoose.Types.ObjectId(message._id) } },
    ...chatMessageCommonAggregation(),
  ]);

  const receivedMessage = messages[0];
  if (!receivedMessage) throw new ApiError(500, "Internal server error");

  // Emit the message to other participants (filtered or original)
  selectedChat.participants.forEach((participant) => {
    if (participant.toString() !== req.user._id.toString()) {
      emitSocketEvent(req, participant.toString(), ChatEventEnum.MESSAGE_RECEIVED_EVENT, receivedMessage);
    }
  });

  // Return the response, indicating the message was saved (whether filtered or original)
  return res.status(201).json(new ApiResponse(201, receivedMessage, "Message saved successfully"));
});



// Delete message
const deleteMessage = asyncHandler(async (req, res) => {
  const { chatId, messageId } = req.params;
  const chat = await Chat.findOne({
    _id: new mongoose.Types.ObjectId(chatId),
    participants: req.user?._id,
  });

  if (!chat) throw new ApiError(404, "Chat does not exist");

  const message = await ChatMessage.findOne({ _id: new mongoose.Types.ObjectId(messageId) });
  if (!message) throw new ApiError(404, "Message does not exist");

  if (message.sender.toString() !== req.user._id.toString()) {
    throw new ApiError(403, "You are not authorized to delete this message");
  }

  if (message.attachments.length > 0) {
    message.attachments.forEach((asset) => removeLocalFile(asset.localPath));
  }

  await ChatMessage.deleteOne({ _id: new mongoose.Types.ObjectId(messageId) });

  if (chat.lastMessage.toString() === message._id.toString()) {
    const lastMessage = await ChatMessage.findOne({ chat: chatId }, {}, { sort: { createdAt: -1 } });
    await Chat.findByIdAndUpdate(chatId, { lastMessage: lastMessage ? lastMessage._id : null });
  }

  chat.participants.forEach((participant) => {
    if (participant.toString() !== req.user._id.toString()) {
      emitSocketEvent(req, participant.toString(), ChatEventEnum.MESSAGE_DELETE_EVENT, message);
    }
  });

  return res.status(200).json(new ApiResponse(200, message, "Message deleted successfully"));
});

// Export all the functions
export { getAllMessages, sendMessage, deleteMessage };