server.js:

```
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const next = require('next');

const dev = process.env.NODE_ENV !== 'production';
const app = next({ dev });
const handle = app.getRequestHandler();

let clients = {}; // maps socket IDs to usernames
let userSockets = {}; // maps usernames to socket IDs

app.prepare().then(() => {
    const server = express();
    const httpServer = createServer(server);
    const io = new Server(httpServer);

    io.on('connection', (socket) => {
        console.log('A user connected:', socket.id);

        socket.on('new-user', (name) => {
            clients[socket.id] = name;
            userSockets[name] = socket.id; // Map username to socket ID
            io.emit('update-user-list', Object.values(clients));
            console.log(`${name} connected with ID: ${socket.id}`);
        });

        socket.on('test-message', (message, toUsername) => {
            const toSocketId = userSockets[toUsername]; // Look up socket ID by username
            if (toSocketId) {
                console.log(`Sending message: "${message}" to ${toUsername}`);
                // to() needs soc-id not username
                socket.to(toSocketId).emit('test-message', message, clients[socket.id]);
            } else {
                console.error(`User ${toUsername} not found`);
            }
        });

        socket.on('disconnect', () => {
            console.log(`${clients[socket.id]} disconnected`);
            delete userSockets[clients[socket.id]]; // Remove the username from the mapping
            delete clients[socket.id];
            io.emit('update-user-list', Object.values(clients));
        });

        // Existing offer, answer, and ICE candidate handlers
        socket.on('offer', (offer, to) => {
            console.log(`Sending offer from ${clients[socket.id]} to ${to}`);
            socket.to(userSockets[to]).emit('offer', offer, socket.id);
        });

        socket.on('answer', (answer, to) => {
            console.log(`Sending answer from ${clients[socket.id]} to ${to}`);
            socket.to(userSockets[to]).emit('answer', answer, socket.id);
        });

        socket.on('ice-candidate', (candidate, to) => {
            socket.to(userSockets[to]).emit('ice-candidate', candidate, socket.id);
        });
    });

    server.all('*', (req, res) => {
        return handle(req, res);
    });

    const PORT = process.env.PORT || 3000;
    httpServer.listen(PORT, (err) => {
        if (err) throw err;
        console.log(`> Ready on http://localhost:${PORT}`);
    });
});

```

page.tsx:
```
"use client";

import { useEffect, useState } from 'react';
import io from 'socket.io-client';

interface FileData {
    fileName: string;
    fileContent: ArrayBuffer | string; // Adjust the type based on how you receive the content
}

const socket = io(); // Connect to the socket.io server

const techPrefixes = [
    "Creative", "Innovative", "Dynamic", "Intelligent", "Agile", "Visionary", "Resourceful"
];

const techNames = [
    "Docker", "Kubernetes", "Python", "Java", "Bootstrap", "Intel", "AMD", "NVIDIA",
    "Tailwind CSS", "CSS", "JavaScript", "HTML", "React", "Vue", "Angular", "Node.js",
    "PostgreSQL", "MongoDB", "GraphQL", "TensorFlow", "PyTorch", "Ruby", "Rust", "Go",
    "Webpack", "VSCode", "Figma", "Git", "Jenkins", "Linux"
];

const generateRandomName = () => {
    const prefix = techPrefixes[Math.floor(Math.random() * techPrefixes.length)];
    const name = techNames[Math.floor(Math.random() * techNames.length)];
    return `${prefix} ${name}`;
};

const Home = () => {
    const [users, setUsers] = useState<string[]>([]);
    const [currentUser, setCurrentUser] = useState<string>('');
    const [selectedPeer, setSelectedPeer] = useState<string | null>(null);
    const [fileInput, setFileInput] = useState<File | null>(null);
    const [peerConnection, setPeerConnection] = useState<RTCPeerConnection | null>(null);
    const [incomingOffers, setIncomingOffers] = useState<string[]>([]);
    const [sendingFiles, setSendingFiles] = useState<string[]>([]);

    useEffect(() => {
        const newUser = generateRandomName();
        setCurrentUser(newUser);
        socket.emit('new-user', newUser);

        console.log(`${newUser} is ready to receive offers.`);

        socket.on('update-user-list', (userList: string[]) => {
            setUsers(userList);
        });

        socket.on('connect', () => {
            console.log('Connected to socket server:', socket.id);
        });

        socket.on('test-message', (message, from) => {
            console.log(`Received message from ${from}:`, message);
        });

        socket.on('offer', (offer, from) => {
            console.log(`Received offer from ${from}:`, offer);
            const acceptOffer = window.confirm(`Receive file from ${from}?`);
            if (acceptOffer) {
                const pc = createPeerConnection();
                pc.setRemoteDescription(new RTCSessionDescription(offer)).then(() => {
                    return pc.createAnswer();
                }).then(answer => {
                    pc.setLocalDescription(answer);
                    socket.emit('answer', answer, from);
                }).catch(error => {
                    console.error('Error handling offer:', error);
                });
            }
        });

        socket.on('ice-candidate', (candidate) => {
            console.log('Received ICE candidate:', candidate);
            if (peerConnection) {
                peerConnection.addIceCandidate(new RTCIceCandidate(candidate))
                    .catch(e => console.error('Error adding ICE candidate:', e));
            }
        });

        return () => {
            socket.off('test-message');
            socket.off('update-user-list');
            socket.off('offer');
            socket.off('ice-candidate');
        };
    }, [peerConnection]);

    const handleFileChange = (event: React.ChangeEvent<HTMLInputElement>) => {
        const files = event.target.files;
        if (files && files.length > 0) {
            setFileInput(files[0]);
        } else {
            setFileInput(null);
        }
    };

    const createPeerConnection = () => {
        const iceServers = [
            { urls: 'stun:stun.l.google.com:19302' } // Google's public STUN server
        ];

        const pc = new RTCPeerConnection({ iceServers });

        pc.onicecandidate = (event) => {
            if (event.candidate) {
                console.log('Sending ICE candidate:', event.candidate);
                socket.emit('ice-candidate', event.candidate, selectedPeer);
            }
        };

        pc.ondatachannel = (event) => {
            const receiveChannel = event.channel;

            receiveChannel.onmessage = (event) => {
                const fileData: FileData = JSON.parse(event.data);
                console.log('Received file:', fileData);
                handleFileReceive(fileData);
            };

            receiveChannel.onopen = () => {
                console.log('Data channel is open on receiver.');
            };

            receiveChannel.onclose = () => {
                console.log('Data channel is closed on receiver.');
            };
        };

        return pc;
    };

    const handleFileReceive = (fileData: FileData) => {
        setIncomingOffers((prev) => [...prev, fileData.fileName]);
        const acceptDownload = window.confirm(`Received file: ${fileData.fileName}. Accept download?`);
        if (acceptDownload) {
            const blob = new Blob([fileData.fileContent], { type: 'application/octet-stream' });
            const url = window.URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = fileData.fileName;
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            window.URL.revokeObjectURL(url);
            console.log('File downloaded:', fileData.fileName);
        }
    };

    const sendFile = () => {
        if (!selectedPeer || !fileInput) {
            alert('Please select a user and a file to send.');
            return;
        }

        const pc = createPeerConnection();
        setPeerConnection(pc);

        const channel = pc.createDataChannel('fileTransfer');

        channel.onopen = () => {
            console.log('Data channel is open.');
            const reader = new FileReader();
            
            reader.onload = (event) => {
                if (event.target && event.target.result !== null) { // Check for null
                    const fileData: FileData = {
                        fileName: fileInput.name,
                        fileContent: event.target.result as ArrayBuffer, // Type assertion
                    };
                    console.log('File data:', fileData);
                    channel.send(JSON.stringify(fileData));
                    setSendingFiles((prev) => [...prev, fileData.fileName]);
                    console.log(`File sent: ${fileData.fileName}`);
                } else {
                    console.error('FileReader target is null or result is null');
                }
            };
            
            reader.readAsArrayBuffer(fileInput);
        };

        channel.onclose = () => {
            console.log('Data channel is closed.');
        };

        channel.onerror = (error) => {
            console.error('Data channel error:', error);
        };

        pc.createOffer()
            .then((offer) => {
                return pc.setLocalDescription(offer);
            })
            .then(() => {
                console.log('Sending offer to peer:', pc.localDescription);
                socket.emit('offer', pc.localDescription, selectedPeer);
            })
            .catch((error) => {
                console.error('Error creating offer:', error);
            });
    };


    const sendTestMessage = () => {
        if (!selectedPeer) {
            alert('Please select a user to send a message.');
            return;
        }

        const message = `hi from ${currentUser}`;
        console.log('Sending message:', message);
        console.log('Selected Peer ID:', selectedPeer);
        socket.emit('test-message', message, selectedPeer);
    };

    useEffect(() => {
        socket.on('answer', (answer) => {
            console.log('Received answer from peer:', answer);
            if (peerConnection) {
                peerConnection.setRemoteDescription(new RTCSessionDescription(answer));
            }
        });

        socket.on('ice-candidate', (candidate) => {
            if (peerConnection) {
                peerConnection.addIceCandidate(new RTCIceCandidate(candidate));
            }
        });

        return () => {
            socket.off('answer');
            socket.off('ice-candidate');
        };
    }, [peerConnection]);

    return (
        <div className="flex flex-col items-center justify-center min-h-screen bg-gradient-to-r from-blue-400 to-purple-600 text-white p-4">
            <h1 className="text-4xl font-bold mb-6">P2P File Share</h1>
            <div className="mb-4">
                <strong>Your ID:</strong> <span className="font-medium">{currentUser}</span>
            </div>

            <div className="flex space-x-4 mb-6 w-full max-w-5xl">
                <div className="flex flex-col w-3/4 border-r-2 border-white pr-4">
                    <h2 className="text-2xl mt-8">Connected Users</h2>
                    <div className="mt-4 space-y-2 overflow-y-auto h-64">
                        {users.map((user) => (
                            user !== currentUser && (
                                <div 
                                    key={user} 
                                    onClick={() => setSelectedPeer(user)} 
                                    className={`cursor-pointer hover:bg-blue-300 rounded p-2 transition duration-200 ${selectedPeer === user ? 'bg-blue-500' : ''}`}
                                >
                                    {user}
                                </div>
                            )
                        ))}
                    </div>

                    <input
                        type="file"
                        onChange={handleFileChange}
                        className="p-2 mb-4 border border-white rounded"
                    />
                    <button 
                        onClick={sendFile} 
                        className="bg-white text-blue-600 font-bold py-2 px-4 rounded transition duration-300 hover:bg-blue-200"
                    >
                        Send File
                    </button>
                    <button 
                        onClick={sendTestMessage} 
                        className="bg-white text-blue-600 font-bold py-2 px-4 rounded transition duration-300 hover:bg-blue-200"
                    >
                        Send Test Message
                    </button>
                </div>

                <div className="flex flex-col w-1/4 pl-4">
                    <h2 className="text-2xl mt-8">Files</h2>
                    <div className="border-b-2 border-white mb-2">
                        <h3 className="font-semibold">Sending</h3>
                        <ul className="mt-2 space-y-2">
                            {sendingFiles.map((file, index) => (
                                <li key={index} className="bg-white text-blue-600 p-2 rounded">
                                    {file}
                                </li>
                            ))}
                        </ul>
                    </div>
                    <div>
                        <h3 className="font-semibold">Receiving</h3>
                        <ul className="mt-2 space-y-2">
                            {incomingOffers.map((file, index) => (
                                <li key={index} className="bg-white text-blue-600 p-2 rounded">
                                    {file}
                                </li>
                            ))}
                        </ul>
                    </div>
                </div>
            </div>
        </div>
    );
};

export default Home;

```
