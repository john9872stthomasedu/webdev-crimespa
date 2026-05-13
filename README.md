 ## Overview
 
This repository contains the starter code and completed project for a **St. Paul Crime Single Page Application (SPA)**, developed as part of a web development course at the **University of St. Thomas**. The project was built collaboratively by Prince Cole, Jack Johnson, and Kaden Stewart.
 
The application allows users to browse, filter, and interact with real crime data from the city of St. Paul through an interactive map interface. Rather than navigating between separate pages, all crime data is dynamically rendered within a single page for a smooth, responsive experience.
 
### Key Features
 
- **Interactive map** powered by the [Leaflet API](https://leafletjs.com/), displaying crime incidents by neighborhood
- **Address search** using the [Nominatim API](https://nominatim.org/) to geocode user-entered locations
- **Crime data integration** via a custom-built St. Paul Crime REST API, supporting both GET (retrieve crimes) and PUT (add a crime) requests
- **Dynamic UI components** built with [Vue 3](https://vuejs.org/) — all HTML, CSS, and JavaScript organized into single-file components
- **Responsive layout** styled with the [Zurb Foundation](https://get.foundation/) CSS framework
### Technologies Used
 
| Tool | Purpose |
|------|---------|
| Vue 3 | Front-end SPA framework |
| Vite | Development server and build tool |
| Leaflet API | Interactive map rendering and neighborhood overlays |
| Nominatim API | Address geocoding from user text input |
| St. Paul Crime API | Custom REST API for querying and adding crime records |
| Foundation CSS | Responsive grid and UI styling |
 
---
 
## Installation Instructions
 
### Prerequisites
 
Make sure the following are installed before getting started:
 
- **Node.js** — version `20.19.0` or `22.12.0+`
  - Download from [nodejs.org](https://nodejs.org/)
- **npm** — included automatically with Node.js
### Steps
 
1. **Clone the repository:**
   ```bash
   git clone https://github.com/john9872stthomasedu/webdev-crimespa.git
   ```
 
2. **Navigate into the project directory:**
   ```bash
   cd webdev-crimespa
   ```
 
3. **Install dependencies:**
   ```bash
   npm install
   ```
 
---
 
## Usage Instructions
 
### Running the Development Server
 
Start the app locally with hot-reloading enabled:
 
```bash
npm run dev
```
 
Then open your browser and navigate to the local URL shown in the terminal (typically `http://localhost:5173`).
 
### Building for Production
 
Compile and bundle the app for deployment:
 
```bash
npm run build
```
 
Output will be placed in the `dist/` folder, ready to be served by any static file host.
 
### Previewing the Production Build
 
Preview the production build locally before deploying:
 
```bash
npm run preview
```
 
### Using the Application
 
- **Browse the map:** The main page displays an interactive Leaflet map of St. Paul with crime data overlaid by neighborhood
- **Search an address:** Use the address search box to jump to a specific location; Nominatim handles converting place names to coordinates
- **View crime details:** Click on neighborhoods or map markers to see crime statistics pulled from the St. Paul Crime API
- **Add a crime:** The app supports submitting new crime records to the database via the API's PUT functionality
- **About page:** Visit `about.html` for team information, a description of the tools used, and a video demonstration

 
