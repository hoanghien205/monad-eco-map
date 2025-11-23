# Dapp Explorer 3D

A stunning modern Dapp discovery interface featuring a 2D list panel on the left and an interactive 3D space on the right — beautifully combining Vuetify + Three.js with a sleek sci-fi/crypto aesthetic.

![Preview](preview.png)
https://monad-eco-map.vercel.app/

## Key Features
- Dapp list with search by name/description + filter by Category
- Interactive 3D space with:
    - Floating spheres with Dapp logos
    - Gorgeous hover & selection effects
    - Glowing ring + pulsing animation when selected
    - Subtle particles around each node
    - Connection lines between same-category Dapps
    - Full OrbitControls (drag to rotate, scroll to zoom)
    - Smooth camera focus on selected Dapp
- Mobile-friendly responsive layout (3D on top on small screens)
- Dark modern UI with gradients & glow
- Clear, readable filter inputs on dark background

## Tech Stack
- Vue 3 + Composition API + `<script setup>`
- Vuetify 3
- Three.js (r162+) + OrbitControls
- Vite (recommended)

## Installation & Run

```bash
git clone https://github.com/yourname/dapp-explorer-3d.git
cd dapp-explorer-3d
npm install    # or yarn
npm run dev    # or yarn dev