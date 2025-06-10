# Baseball Statistics Viewer

A Vue.js application that displays and analyzes MLB at-bat statistics from April 1-14, 2018. The application provides an interactive table with filtering, sorting, and pagination capabilities.

## Features

- Interactive data table with MLB at-bat statistics
- Filter by batter name, pitcher name, and play outcome
- Sort by any column
- Pagination for better data navigation
- Video links for select plays
- Responsive design
- Collapsible explanation section with legend

## Prerequisites

- Node.js (v16 or higher)
- npm (v7 or higher)

## Installation

1. Clone the repository:

```bash
git clone <repository-url>
cd baseball-stats-viewer
```

2. Install dependencies:

```bash
npm install
```

## Development

To start the development server:

```bash
npm run dev
```

This will start the application in development mode with hot-reload enabled. The application will be available at `http://localhost:5173`.

## Building for Production

To create a production build:

```bash
npm run build
```

The build artifacts will be stored in the `dist/` directory.

To preview the production build locally:

```bash
npm run preview
```

## Code Quality

The project uses ESLint and Prettier for code quality and formatting:

```bash
# Run ESLint
npm run lint

# Format code with Prettier
npm run format
```

## Project Structure

```
├── public/
│   ├── data.csv        # Baseball statistics data
│   └── legend.csv      # Data legend
├── src/
│   ├── components/
│   │   ├── BaseballTable.vue    # Main table component
│   │   └── icons/              # SVG icons
│   ├── App.vue                 # Root component
│   └── main.js                 # Application entry point
├── index.html
├── package.json
├── vite.config.js
└── README.md
```

## Data Format

The application uses a CSV file (`data.csv`) containing the following columns:

- BATTER: Batter's name
- PITCHER: Pitcher's name
- GAME_DATE: Date of the game
- LAUNCH_ANGLE: Vertical angle of the ball leaving the bat
- EXIT_SPEED: Velocity of the ball leaving the bat
- HIT_DISTANCE: Distance the ball traveled
- PLAY_OUTCOME: Result of the play
- VIDEO_LINK: URL to video of the play (if available)

## Technologies Used

- Vue.js 3
- Vite
- PapaParse (for CSV parsing)
- ESLint
- Prettier

## Browser Support

The application is built with modern web technologies and supports:

- Chrome (latest)
- Firefox (latest)
- Safari (latest)
- Edge (latest)

## License

[Add your license information here]
