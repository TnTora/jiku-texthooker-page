# jiku-texthooker-page

 Web interface to receive text via websocket. This is a stripped down standalone version from the one included in [Jiku](https://github.com/TnTora/Jiku).

 Main features:

 - Horizontal and vertical text supported
 - Support for multiple presets each storing lines and options separately 
 - virtual list render only a fraction of the lines at a time, allowing great performance even with thousends of lines stored
 - line counter and current top line indicator
 - automatic scrolling on new line if near list bottom
 - set maxium number of successive line repetition
 - customise font, font-size, line-height


Possible future features:

- clipboard monitoring
- edit line text
- add option to remove text between parentheses 
- include the text parsing and anki integration from [Jiku](https://github.com/TnTora/Jiku) in a simplified form


## Developing

Install dependencies with `npm install` (or `pnpm install` or `yarn`), start a development server:

```sh
npm run dev

# or start the server and open the app in a new browser tab
npm run dev -- --open
```

## Building

To build into a single html file:

```sh
npm run build
```

You can preview the production build with `npm run preview`.


## Acknowledgement

[Texthooker UI](https://github.com/Renji-XD/texthooker-ui)

