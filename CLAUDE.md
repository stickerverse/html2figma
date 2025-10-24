# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

html2figma is a web-to-Figma conversion tool consisting of two integrated components:
1. **Chrome Extension** - Captures web pages and extracts their structure, styles, and assets
2. **Figma Plugin** - Imports the extracted data into Figma as editable designs

## Build and Development Commands

### Chrome Extension (`chrome-extension/`)
```bash
cd chrome-extension
npm install
npm run build          # Production build with webpack
npm run dev            # Development mode with watch
npm run type-check     # TypeScript type checking
```

### Figma Plugin (`chrome-extension/figma-plugin/`)
```bash
cd chrome-extension/figma-plugin
npm install
npm run build          # Compile TypeScript (main + UI)
npm run watch          # Watch mode for development
```

## Architecture

### Data Flow
1. User triggers capture via Chrome Extension popup
2. **Content Script** (`content-script.ts`) coordinates the extraction
3. **Injected Script** (`injected-script.ts`) runs in page context and extracts DOM
4. Extraction utilities process the page into a `WebToFigmaSchema` object
5. User exports schema and imports into Figma Plugin
6. **Figma Plugin** (`code.ts`) recreates the design in Figma

### Chrome Extension Components

#### Core Scripts
- **background.ts** - Service worker, manages extension lifecycle and message routing
- **content-script.ts** - Injected into pages, coordinates extraction by injecting the extraction script
- **injected-script.ts** - Runs in page context, orchestrates `PageExtractor` to build the schema

#### Extraction Utilities (`chrome-extension/src/utils/`)
- **dom-extractor.ts** - Core extraction engine; recursively traverses DOM and converts elements to `ElementNode` objects. Handles layout, styles, auto-layout detection, text/image/vector nodes, and pseudo-elements
- **style-parser.ts** - Converts CSS computed styles to Figma-compatible format (fills, strokes, effects, text styles, corner radius)
- **component-detector.ts** - Identifies reusable components based on class patterns and structure
- **variants-collector.ts** - Collects different states of interactive elements (hover, focus, active)
- **state-capturer.ts** - Captures element states by simulating interactions
- **asset-handler.ts** - Registers and manages images and SVGs in the asset registry

#### Schema (`chrome-extension/src/types/schema.ts`)
Central data structure shared between extension and plugin:
- **WebToFigmaSchema** - Top-level schema containing metadata, tree, assets, styles, components, variants
- **ElementNode** - Represents a single design element with layout, styling, children, and metadata
- **Registries** - AssetRegistry, StyleRegistry, ComponentRegistry, VariantsRegistry for deduplication and organization

### Figma Plugin Components

#### Core Files
- **code.ts** - Main plugin entry point; receives schema from UI, loads fonts, creates main frame
- **node-builder.ts** - Builds Figma nodes from `ElementNode` objects (frames, text, images, rectangles, vectors)
- **importer.ts** - Handles import logic and data validation
- **design-system-builder.ts** - Creates design system elements (color styles, text styles)
- **component-manager.ts** - Creates and manages Figma components
- **style-manager.ts** - Creates and applies Figma styles
- **variants-frame-builder.ts** - Builds component variant frames
- **ui/ui.ts** - Plugin UI for import options and progress

## Key Implementation Details

### Node Type Determination
The system maps HTML elements to Figma node types:
- `IMG`, `PICTURE` → `IMAGE`
- `SVG` elements → `VECTOR`
- Text-only elements → `TEXT`
- Block elements → `FRAME`
- Elements with background images but no children → `RECTANGLE`

Logic in `dom-extractor.ts:determineNodeType()`

### Auto Layout Detection
CSS flexbox is converted to Figma auto-layout:
- `display: flex` → Auto-layout frame
- `flex-direction` → `layoutMode` (HORIZONTAL/VERTICAL)
- `justify-content` → `primaryAxisAlignItems`
- `align-items` → `counterAxisAlignItems`
- `gap` → `itemSpacing`
- Padding values mapped directly

Logic in `dom-extractor.ts:extractAutoLayout()`

### Semantic Naming
Elements are named based on priority:
1. `aria-label` attribute
2. `data-testid` attribute
3. `id` attribute
4. First non-utility CSS class (filters out Tailwind-style utilities)
5. HTML tag name

Logic in `dom-extractor.ts:generateSemanticName()`

### Style Registries
Styles are deduplicated and tracked:
- Colors keyed by hex value with usage count
- Text styles keyed by `fontFamily-fontWeight-fontSize`
- Effects hashed by JSON stringification

## Testing and Debugging

### Chrome Extension
1. Run `npm run dev` in watch mode
2. Load unpacked extension from `chrome-extension/dist/`
3. Check browser console and extension DevTools for errors
4. Inspect `WebToFigmaSchema` output in popup before export

### Figma Plugin
1. Run `npm run watch` for auto-recompilation
2. In Figma: Plugins > Development > Import plugin from manifest
3. Select `chrome-extension/figma-plugin/manifest.json`
4. Use Figma DevTools (Plugins > Development > Open Console) for debugging
5. Check `stats` object in `code.ts` for import statistics

## Common Development Patterns

### Adding New Style Properties
1. Extend relevant interface in `schema.ts` (e.g., `ElementNode`, `Fill`, `Effect`)
2. Add extraction logic in `style-parser.ts`
3. Add application logic in `node-builder.ts` or `code.ts`

### Adding New Node Types
1. Add type to `ElementNode['type']` in `schema.ts`
2. Update `determineNodeType()` in `dom-extractor.ts`
3. Add extraction method in `dom-extractor.ts` (e.g., `extractVectorNode()`)
4. Add builder method in `code.ts` or `node-builder.ts`

### Modifying Capture Options
1. Update `CaptureOptions` interface in `schema.ts`
2. Update default options in `background.ts:onInstalled`
3. Add UI controls in `popup.ts` and `popup.html`
4. Implement feature in extraction utilities (`injected-script.ts` or relevant utility)
5. Handle in Figma plugin if needed (`code.ts`)

## File Organization

```
chrome-extension/
├── src/
│   ├── background.ts                 # Service worker
│   ├── content-script.ts             # Coordination layer
│   ├── injected-script.ts            # Page-context extraction
│   ├── popup/                        # Extension UI
│   ├── utils/                        # Extraction utilities
│   │   ├── dom-extractor.ts          # Core DOM extraction
│   │   ├── style-parser.ts           # Style conversion
│   │   ├── component-detector.ts     # Component detection
│   │   ├── variants-collector.ts     # State variants
│   │   ├── state-capturer.ts         # State simulation
│   │   └── asset-handler.ts          # Asset management
│   └── types/
│       └── schema.ts                 # Shared type definitions
├── figma-plugin/
│   ├── src/
│   │   ├── code.ts                   # Main plugin code
│   │   ├── node-builder.ts           # Node construction
│   │   ├── importer.ts               # Import logic
│   │   ├── design-system-builder.ts  # Design system
│   │   ├── component-manager.ts      # Component management
│   │   ├── style-manager.ts          # Style management
│   │   └── variants-frame-builder.ts # Variant frames
│   └── ui/
│       └── ui.ts                     # Plugin UI
├── webpack.config.js                 # Extension bundler
├── package.json                      # Extension dependencies
└── manifest.json                     # Extension manifest

chrome-extension/figma-plugin/
├── package.json                      # Plugin dependencies
├── manifest.json                     # Figma plugin manifest
├── tsconfig.json                     # Main code config
└── tsconfig.ui.json                  # UI code config
```

## Important Notes

- The extension and plugin are separate build artifacts with separate dependency trees
- `schema.ts` must remain synchronized between extension and plugin (it's currently only in extension)
- Font loading in Figma requires fonts to be available in the user's Figma account
- The plugin currently creates placeholder rectangles for images (image import requires additional implementation)
- Coordinate systems: Extension captures in viewport coordinates, plugin uses relative positioning
