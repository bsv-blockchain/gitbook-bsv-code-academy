# Assets

This directory contains images, diagrams, and other media files used throughout the BSV Code Academy documentation.

## Organization

Assets are organized by type and context:

```
assets/
├── diagrams/          # Architecture and flow diagrams
├── screenshots/       # UI screenshots and examples
├── logos/            # BSV and related logos
├── icons/            # Icons and small graphics
└── examples/         # Example images and media
```

## File Naming Convention

Use descriptive, kebab-case names:
- `transaction-flow-diagram.png`
- `wallet-setup-screenshot.png`
- `bsv-logo-horizontal.svg`
- `overlay-architecture.png`

## Supported Formats

- **Images**: PNG, JPG, SVG
- **Diagrams**: SVG (preferred), PNG
- **Animations**: GIF, WebP

## Usage in Documentation

Reference assets using relative paths:

```markdown
![Transaction Flow](../../assets/diagrams/transaction-flow-diagram.png)
```

## Image Guidelines

1. **Size**: Keep images under 1MB when possible
2. **Resolution**: Use 2x resolution for retina displays
3. **Format**:
   - Use SVG for diagrams and logos
   - Use PNG for screenshots
   - Use JPG for photos
4. **Alt Text**: Always provide descriptive alt text

## Creating Diagrams

### Recommended Tools

- [Excalidraw](https://excalidraw.com/) - Hand-drawn style diagrams
- [Draw.io](https://draw.io/) - Technical diagrams
- [Mermaid](https://mermaid.js.org/) - Code-based diagrams
- [Figma](https://figma.com/) - Professional designs

### Diagram Templates

Common diagram types needed:
- Transaction flow diagrams
- Network topology diagrams
- System architecture diagrams
- Sequence diagrams
- State machine diagrams

## Example Assets

### Transaction Flow Diagram
Shows the complete lifecycle of a BSV transaction from creation to confirmation.

### Overlay Network Architecture
Illustrates how overlay networks sit on top of the BSV blockchain.

### Key Derivation Diagram
Visualizes BRC-42 key derivation paths.

### UTXO Model Diagram
Explains the UTXO model vs account model.

## Contributing Assets

When adding new assets:
1. Ensure you have rights to use the asset
2. Optimize file size before committing
3. Place in appropriate subdirectory
4. Use descriptive file names
5. Update this README if adding new categories

## Asset Optimization

### Images
```bash
# Optimize PNG
pngquant image.png --output image-optimized.png

# Optimize JPG
jpegoptim --max=85 image.jpg

# Optimize SVG
svgo image.svg
```

### Batch Optimization
```bash
# Install tools
npm install -g svgo pngquant-bin jpegoptim-bin

# Optimize all PNGs
find . -name "*.png" -exec pngquant --ext .png --force {} \;

# Optimize all SVGs
find . -name "*.svg" -exec svgo {} \;
```

## Copyright and Licensing

- BSV logos: Used with permission from BSV Association
- Diagrams: Created for this project (MIT License)
- Third-party images: Properly attributed with licenses

## Placeholder Assets

For documentation in progress, use placeholder images:
- https://via.placeholder.com/800x400.png?text=Diagram+Coming+Soon

Replace with actual assets before publication.
