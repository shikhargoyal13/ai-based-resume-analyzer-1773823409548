// FILE: package.json
{
  "name": "ai-resume-analyzer",
  "version": "1.0.0",
  "description": "AI-Based Resume Analyzer",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.17.1",
    "mongodb": "^3.6.4",
    "mongoose": "^5.12.3",
    "body-parser": "^1.19.0",
    "cors": "^2.8.5",
    "dotenv": "^10.0.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.7"
  }
}

// FILE: frontend/package.json
{
  "name": "ai-resume-analyzer-frontend",
  "version": "1.0.0",
  "description": "AI-Based Resume Analyzer Frontend",
  "main": "index.js",
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test"
  },
  "dependencies": {
    "react": "^17.0.2",
    "react-dom": "^17.0.2",
    "react-scripts": "4.0.3",
    "axios": "^0.21.1",
    "react-router-dom": "^5.2.0"
  },
  "devDependencies": {
    "eslint": "^7.25.0",
    "prettier": "^2.3.2"
  }
}

// FILE: .env.example
MONGO_URI=mongodb://localhost:27017/resume-analyzer
AI_ENGINE_URL=http://localhost:8000
CLOUD_PLATFORM=AWS

// FILE: frontend/.env.example
REACT_APP_API_URL=http://localhost:3000
REACT_APP_AI_ENGINE_URL=http://localhost:8000

// FILE: server.js
const express = require('express');
const app = express();
const cors = require('cors');
const mongoose = require('mongoose');
const multer = require('multer');
const dotenv = require('dotenv');
const resumeController = require('./controllers/resumeController');
const analysisController = require('./controllers/analysisController');
const errorMiddleware = require('./middleware/errorMiddleware');

dotenv.config();

const port = process.env.PORT || 3000;

app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, './uploads');
  },
  filename: (req, file, cb) => {
    cb(null, file.originalname);
  }
});

const upload = multer({ storage: storage });

mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => {
    console.log('Connected to MongoDB');
  })
  .catch((err) => {
    console.error('Error connecting to MongoDB:', err);
  });

app.post('/upload', upload.single('resume'), resumeController.uploadResume);
app.get('/results', analysisController.getAnalysisResults);

app.use(errorMiddleware);

app.listen(port, () => {
  console.log(`Server listening on port ${port}`);
});

// FILE: controllers/resumeController.js
const mongoose = require('mongoose');
const Resume = require('../models/Resume');

exports.uploadResume = async (req, res) => {
  try {
    const resume = new Resume({
      fileName: req.file.originalname,
      fileContent: req.file.buffer
    });
    await resume.save();
    res.json({ message: 'Resume uploaded successfully', resumeId: resume._id });
  } catch (err) {
    res.status(500).json({ message: 'Error uploading resume', error: err });
  }
};

// FILE: controllers/analysisController.js
const mongoose = require('mongoose');
const AnalysisResult = require('../models/AnalysisResult');
const axios = require('axios');

exports.getAnalysisResults = async (req, res) => {
  try {
    const resumeId = req.query.resumeId;
    const analysisResult = await AnalysisResult.findOne({ resumeId: resumeId });
    if (!analysisResult) {
      const resume = await mongoose.model('Resume').findOne({ _id: resumeId });
      if (!resume) {
        res.status(404).json({ message: 'Resume not found' });
      } else {
        const response = await axios.post(process.env.AI_ENGINE_URL, {
          resume: resume.fileContent.toString()
        });
        const analysisResult = new AnalysisResult({
          resumeId: resumeId,
          skillsMatch: response.data.skillsMatch,
          improvements: response.data.improvements,
          score: response.data.score
        });
        await analysisResult.save();
        res.json({ skillsMatch: analysisResult.skillsMatch, improvements: analysisResult.improvements, score: analysisResult.score });
      }
    } else {
      res.json({ skillsMatch: analysisResult.skillsMatch, improvements: analysisResult.improvements, score: analysisResult.score });
    }
  } catch (err) {
    res.status(500).json({ message: 'Error getting analysis results', error: err });
  }
};

// FILE: models/Resume.js
const mongoose = require('mongoose');

const resumeSchema = new mongoose.Schema({
  fileName: String,
  fileContent: Buffer
});

module.exports = mongoose.model('Resume', resumeSchema);

// FILE: models/AnalysisResult.js
const mongoose = require('mongoose');

const analysisResultSchema = new mongoose.Schema({
  resumeId: String,
  skillsMatch: Array,
  improvements: Array,
  score: Number
});

module.exports = mongoose.model('AnalysisResult', analysisResultSchema);

// FILE: middleware/errorMiddleware.js
const errorHandler = (err, req, res, next) => {
  console.error(err);
  res.status(500).json({ message: 'Internal Server Error', error: err });
};

module.exports = errorHandler;

// FILE: frontend/src/index.js
import React from 'react';
import ReactDOM from 'react-dom';
import { BrowserRouter } from 'react-router-dom';
import App from './App';

ReactDOM.render(
  <BrowserRouter>
    <App />
  </BrowserRouter>,
  document.getElementById('root')
);

// FILE: frontend/src/App.js
import React from 'react';
import { Route, Switch } from 'react-router-dom';
import Home from './pages/Home';
import UploadResume from './pages/UploadResume';
import AnalysisResults from './pages/AnalysisResults';
import NavigationBar from './components/NavigationBar';

function App() {
  return (
    <div>
      <NavigationBar />
      <Switch>
        <Route path="/" exact component={Home} />
        <Route path="/upload-resume" component={UploadResume} />
        <Route path="/analysis-results" component={AnalysisResults} />
      </Switch>
    </div>
  );
}

export default App;

// FILE: frontend/src/pages/Home.js
import React from 'react';

function Home() {
  return (
    <div>
      <h1>Welcome to Resume Analyzer</h1>
      <p>This application allows you to upload your resume and get analysis results.</p>
    </div>
  );
}

export default Home;

// FILE: frontend/src/pages/UploadResume.js
import React, { useState } from 'react';
import axios from 'axios';

function UploadResume() {
  const [file, setFile] = useState(null);
  const [uploading, setUploading] = useState(false);
  const [resumeId, setResumeId] = useState(null);

  const handleFileChange = (event) => {
    setFile(event.target.files[0]);
  };

  const handleUpload = async () => {
    setUploading(true);
    const formData = new FormData();
    formData.append('resume', file);
    try {
      const response = await axios.post('http://localhost:5000/upload', formData);
      setResumeId(response.data.resumeId);
    } catch (error) {
      console.error(error);
    } finally {
      setUploading(false);
    }
  };

  return (
    <div>
      <h1>Upload Resume</h1>
      <input type="file" onChange={handleFileChange} />
      <button onClick={handleUpload} disabled={uploading}>
        {uploading ? 'Uploading...' : 'Upload'}
      </button>
      {resumeId && <p>Resume uploaded successfully. ID: {resumeId}</p>}
    </div>
  );
}

export default UploadResume;

// FILE: frontend/src/pages/AnalysisResults.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

function AnalysisResults() {
  const [results, setResults] = useState(null);

  useEffect(() => {
    const fetchResults = async () => {
      try {
        const response = await axios.get('http://localhost:5000/results');
        setResults(response.data);
      } catch (error) {
        console.error(error);
      }
    };
    fetchResults();
  }, []);

  return (
    <div>
      <h1>Analysis Results</h1>
      {results ? (
        <ul>
          {results.map((result, index) => (
            <li key={index}>{result}</li>
          ))}
        </ul>
      ) : (
        <p>Loading...</p>
      )}
    </div>
  );
}

export default AnalysisResults;

// FILE: frontend/src/components/NavigationBar.js
import React from 'react';
import { Link } from 'react-router-dom';

function NavigationBar() {
  return (
    <nav>
      <ul>
        <li>
          <Link to="/">Home</Link>
        </li>
        <li>
          <Link to="/upload-resume">Upload Resume</Link>
        </li>
        <li>
          <Link to="/analysis-results">Analysis Results</Link>
        </li>
      </ul>
    </nav>
  );
}

export default NavigationBar;

// FILE: frontend/src/services/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: 'http://localhost:5000',
});

export const uploadResume = async (file) => {
  const formData = new FormData();
  formData.append('resume', file);
  try {
    const response = await api.post('/upload', formData);
    return response.data;
  } catch (error) {
    console.error(error);
  }
};

export const getAnalysisResults = async () => {
  try {
    const response = await api.get('/results');
    return response.data;
  } catch (error) {
    console.error(error);
  }
};

// FILE: frontend/src/App.css
body {
  font-family: Arial, sans-serif;
}

nav {
  background-color: #333;
  color: #fff;
  padding: 1em;
  text-align: center;
}

nav ul {
  list-style: none;
  margin: 0;
  padding: 0;
}

nav li {
  display: inline-block;
  margin-right: 20px;
}

nav a {
  color: #fff;
  text-decoration: none;
}

nav a:hover {
  color: #ccc;
}

// FILE: package.json
{
  "name": "resume-analyzer",
  "version": "1.0.0",
  "scripts": {
    "start": "concurrently \"npm run start:backend\" \"npm run start:frontend\"",
    "install-all": "npm install && npm run install:frontend",
    "start:backend": "node backend/index.js",
    "start:frontend": "npm run --prefix frontend start",
    "install:frontend": "npm run --prefix frontend install"
  },
  "dependencies": {
    "concurrently": "^7.0.0",
    "express": "^4.17.1",
    "mongoose": "^6.2.10"
  }
}

// FILE: .gitignore
node_modules
frontend/node_modules
backend/uploads

// FILE: docker-compose.yml
version: '3'
services:
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    depends_on:
      - mongo
    environment:
      - DATABASE_URL=mongodb://mongo:27017/resume-analyzer
    volumes:
      - ./backend/uploads:/app/uploads
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend
    environment:
      - CHOKIDAR_USEPOLLING=true
    volumes:
      - ./frontend:/app
  mongo:
    image: mongo:latest
    volumes:
      - mongo-data:/data/db
volumes:
  mongo-data:

// FILE: README.md
This is a Resume Analyzer application. 
To install all dependencies, run `npm run install-all` in the root directory. 
To start the application, run `npm start` in the root directory. 
The backend server will be available at `http://localhost:5000` and the frontend will be available at `http://localhost:3000`. 
To upload a resume, navigate to `http://localhost:3000/upload-resume` and select a file. 
The analysis results will be displayed at `http://localhost:3000/analysis-results`. 
To use the application with Docker, run `docker-compose up` in the root directory. 
The application will be available at the same URLs as above. 
To stop the Docker containers, run `docker-compose down` in the root directory.