# Placeholder Forge

Placeholder Forge is a web-based tool for generating colorful placeholder images with optional randomized EXIF metadata. It allows users to create images with custom dimensions, background colors, and EXIF data, and download them individually or as a ZIP archive.

## Features

- Generate placeholder images with customizable dimensions and formats (JPEG/PNG).
- Add randomized EXIF metadata to JPEG images, including GPS and device information.
- Preview generated images in a gallery (list or tile view).
- Download images individually or as a ZIP archive.
- Fully client-side: no data is sent to any server.

## Technologies Used

- **Vue 3** with the Composition API for reactive state management.
- **Nuxt.js** for streamlined development and deployment.
- **JSZip** for creating ZIP archives.
- **piexifjs** for injecting EXIF metadata into JPEG images.
- **faker.js** for generating random EXIF metadata.
- **Tailwind CSS** for styling.

## Setup

Make sure to install the dependencies:

```bash
# npm
npm install

# pnpm
pnpm install

# yarn
yarn install

# bun
bun install
```

## Development Server

Start the development server on `http://localhost:3000`:

```bash
# npm
npm run dev

# pnpm
pnpm run dev

# yarn
yarn dev

# bun
bun run dev
```

## Usage

1. Open the application in your browser.
2. Configure the settings:
   - Number of images to generate.
   - Random or fixed dimensions.
   - Image format (JPEG/PNG).
   - Optional EXIF metadata for JPEG images.
   - Background color.
3. Click **Generate** to create the images.
4. Preview the generated images in the gallery.
5. Download individual images or export all as a ZIP archive.

## Disclaimer

Placeholder Forge runs entirely in your browser. Images are painted using the HTML5 canvas, and optional EXIF metadata is injected using `piexifjs`. No files or data ever leave your device.

- **Note**: EXIF metadata is only supported for JPEG images. PNG outputs will not include EXIF data.

## License

This project is licensed under the [MIT License](LICENSE).

## Contact

For questions or feedback, please open an issue in the repository.