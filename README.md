1. Archirecure diagram
    - > ![alt text](image.png)
2. Initialize the Frontend
    - > npx create-next-app@latest frontend --typescript --tailwind --app --no-src-dir
2.  Install Required Dependencies(frontend)
    - > npm install lucide-react
3. Fix Tailwind v4 Configuration:Next.js 15.5 comes with Tailwind CSS v4, which has a different configuration approach. We need to update two files:
    - > export default {
    plugins: {
        '@tailwindcss/postcss': {},
    },
}


## Step 1: Setup the Backend Server

1.  Start the Backend Server
    - > cd backend
    - > uv init --bare
    - > uv python pin 3.14
    - > uv add -r requirements.txt (run this command after adding all the dependencies inside requirements.txt)
    - > uv run uvicorn server:app --reload

## Step 2: Start the Frontend Development Server

1. Start frontend
    - > cd frontend
    - > npm run dev

## Step 3: Start/Restart Backend Server

1. Stop the backend server (Ctrl+C in the terminal) and restart it:
    - > uv run uvicorn server:app --reload


## Set Up AWS Environment

Follow Week4.md File
