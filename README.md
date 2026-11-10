# terminal-analisis-financiero

Terminal local para revisar el oro (XAU/USD) y el ETF SPY. En la interfaz el proyecto se llama **Nexus** (NEXUS V3 en el panel izquierdo): un backend con FastAPI y un frontend con React que corren en tu computadora.

No envía órdenes a un bróker. Las operaciones y los aportes que anotas se quedan en el navegador.

## Qué incluye

La aplicación React tiene tres pestañas.

### XAU/USD Terminal

Tres columnas.

**Gráfico.** Velas con TradingView Lightweight Charts. Arriba puedes elegir la temporalidad: 5m, 15m, 30m o 1h (la que queda guardada arranca en 15m). Sobre las velas se dibuja una EMA de 50 períodos, un histograma de volumen y flechas `BUY IDEA` o `SELL IDEA` cuando, en esa vela, coinciden la tendencia, el RSI y un rango mayor que el promedio reciente.

El oro sale de la API pública de Binance, símbolo **PAXGUSDT**, con un máximo de 500 velas. El componente también tiene botones para **Oro (XAU/USD)** y **S&P 500 (SPY)** y, si el activo es SPY, pide `/api/market/spy/history`. El store no declara `activeAsset` ni `setActiveAsset`, así que al abrir la pestaña la petición va a oro. El precio de SPY que sí puedes consultar a mano está en la pestaña del portafolio.

Las velas se piden al abrir la pestaña y otra vez si cambias la temporalidad o el desplazamiento de precio. No hay un temporizador que las refresque solo.

**Sincronizar con TradingView.** Si el precio de tu bróker no coincide con el feed, escribes el precio que ves en TradingView y pulsas **Sincronizar**. Se calcula un desplazamiento en dólares (offset) y se suma al OHLC que usan el motor de confluencia y el precio que se manda a Gemini. Las velas del gráfico se dibujan con el feed sin ese desplazamiento. El offset no se guarda: al recargar la página vuelve a cero.

**Risk Engine.** Con el capital de la cuenta, el porcentaje máximo de riesgo y el stop loss en pips calcula:

- el monto en riesgo: `capital × riesgo% / 100`
- el tamaño en lotes: ese monto dividido entre `(pips de stop × 10)`

El `10` es un valor fijo de pip para un lote estándar de XAU/USD. Los valores iniciales son 100000 de capital, 1% de riesgo y 50 pips. Esos tres campos se guardan en el navegador.

**Motor de confluencia** (el recuadro *Strict Engine*). Se recalcula en el navegador cuando ya hay al menos 50 velas en memoria:

- Tendencia: el cierre está sobre el promedio de los últimos 50 cierres y ese promedio está sobre el de los últimos 100 (compra), o al revés (venta). La etiqueta dice EMA 50/100; la cuenta es un promedio simple.
- Momentum: RSI de 14. En tendencia de compra pide un RSI entre 30 y 55; en venta, entre 45 y 70.
- Volatilidad: el rango de la última vela (máximo menos mínimo) supera 1.1 veces el promedio de los rangos de las últimas 14 velas.
- Risk/reward: se marca cuando las tres condiciones de arriba se cumplen a la vez.

Con las cuatro marcas activas, el recuadro muestra `BUY CONFIRMED` o `SELL CONFIRMED`. Si falta alguna, se queda en `WAIT FOR SETUP`.

**Active Operations.** Puedes anotar una compra o una venta con precio de entrada y lotes (el campo de lotes empieza en 0.01). El resultado en dólares es la diferencia entre la entrada y el último cierre cargado, multiplicada por los lotes y por 100. Cada fila tiene **Eliminar**. La lista queda en el navegador.

**Macro AI Intelligence.** Pegas tu clave de Gemini y pulsas **Scan Global Macros**. El backend prueba, en este orden, `gemini-2.0-flash`, `gemini-flash-latest`, `gemini-pro-latest`, `gemini-1.5-flash` y `gemini-pro`, y usa la primera que responda. El pedido lleva el activo, el precio, el RSI, la señal del motor y las operaciones abiertas. El prompt pide una recomendación táctica (Hold, Close, BE o Add), en HTML y con un máximo de 100 palabras. Ese HTML se muestra en el panel. Sin clave, el panel avisa que hace falta. La clave se guarda en el navegador y el backend solo la recibe en esa petición.

### My TradingView

Embebe el widget de gráfico avanzado de TradingView: símbolo inicial `OANDA:XAUUSD`, intervalo de 15 minutos, tema oscuro e idioma español. La configuración deja cambiar de símbolo, usar rangos de fechas y mostrar detalles, hotlist y calendario. La publicación del widget queda habilitada y el guardado de imagen queda apagado. El texto de la pestaña indica que, desde el menú del propio gráfico, puedes iniciar sesión en TradingView para ver tus análisis. Esa sesión la maneja TradingView, no este backend.

### SP500 Portfolio

Seguimiento local de aportes (DCA) al ETF SPY.

- **Actualizar Precio en Vivo** llama a `GET /api/market/spy` y pone el balance en acciones por ese precio.
- **Añadir Aporte** pide un monto en USD. El precio de SPY es opcional: si lo escribes, las acciones de ese aporte salen de `monto / ese precio`; si lo dejas vacío, de `monto / último precio consultado`.
- En pantalla ves el capital invertido, las acciones acumuladas (cuatro decimales) y el balance. El balance va en verde si es mayor o igual que el capital invertido, y en rojo si es menor.
- El historial lista fecha, inversión, precio SPY y acciones. **Eliminar** resta el monto y las acciones de esa fila.

El portafolio, el riesgo, la clave de Gemini, las operaciones y la temporalidad se guardan en el navegador bajo la clave `nexus-v3-storage`.

## API

| Método | Ruta | Qué responde |
| --- | --- | --- |
| `GET` | `/api/market/gold` | Velas de PAXGUSDT. `interval` sale en `15m` si no lo envías. `period` se acepta y no entra en la llamada a Binance. El tope es 500 velas. |
| `GET` | `/api/market/spy` | Último cierre de SPY del día, en intervalo de 1 minuto. Si yfinance devuelve una tabla vacía, responde `{"price": 510.50}`. |
| `GET` | `/api/market/spy/history` | Apertura, máximo, mínimo, cierre y volumen de SPY. `interval` por defecto `15m` y `period` por defecto `5d`. El gráfico manda `1mo`, o `3mo` cuando la temporalidad es 1h. |
| `POST` | `/api/ai/analyze` | Análisis de Gemini. El cuerpo pide `api_key` y `price`. Opcionales: `rsi` (0), `trend` (`Neutral`), `trades` (lista vacía) y `asset` (`XAU/USD`). Sin clave responde 400. |

CORS acepta cualquier origen. Si una consulta de mercado o de Gemini falla, la ruta responde 500 con el detalle del error.

Al arrancar, el backend crea `nexus.db` con la tabla `portfolio_tx` (`amount_usd`, `spy_price`, `shares`, `timestamp`). Ninguna ruta lee ni escribe esa tabla. El archivo está en `.gitignore`. El portafolio que ves en la interfaz no pasa por esa base.

## Stack

**Frontend** (`nexus-terminal-v3` en `frontend/package.json`)

- React 18
- Vite 5
- Tailwind CSS 3
- Zustand 4, con persistencia en `localStorage`
- Axios
- TradingView Lightweight Charts 4

Chart.js figura en `package.json`. La aplicación React no lo importa.

**Backend** (`backend/requirements.txt`)

- FastAPI
- Uvicorn
- yfinance
- Pydantic
- google-generativeai
- pysqlite3, declarado en los requisitos; `main.py` usa `sqlite3` de la biblioteca estándar de Python

El oro no pasa por yfinance. Se lee con `urllib` desde `https://api.binance.com/api/v3/klines`.

## Cómo instalarlo y ejecutarlo

Hace falta Python 3, Node.js con npm, y salida a internet hacia Binance, Yahoo Finance, la API de Gemini y el widget de TradingView.

El frontend llama fijo a `http://localhost:8000`. Deja el backend en ese puerto. Vite sirve la interfaz en `http://localhost:5173`.

### Windows

1. Doble clic en `INSTALACION_INICIAL.bat`. Crea `backend/venv`, instala `requirements.txt`, corre `npm install` dentro de `frontend` y al final ejecuta `git pull origin main`.
2. Doble clic en `ARRANCAR_TODO.bat`. Levanta el backend con `uvicorn main:app --reload`, el frontend con `npm run dev`, espera 3 segundos y abre `http://localhost:5173`.
3. `INICIAR_NEXUS.bat` arranca los dos procesos igual y no abre el navegador.

Mientras uses Nexus, no cierres las ventanas de consola.

### Linux o macOS

Backend, en una terminal:

```bash
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload
```

Frontend, en otra terminal:

```bash
cd frontend
npm install
npm run dev
```

Entra a [http://localhost:5173](http://localhost:5173).

`npm run build` deja el estático en `frontend/dist`. `npm run preview` sirve esa carpeta.

En Windows el entorno virtual se activa con `venv\Scripts\activate`, que es lo que usan los `.bat`.

### Página estática de la raíz

El `index.html` de la raíz no es el de Vite y los `.bat` no lo abren. Si lo abres directo en el navegador verás una página anterior: precios de XAU escritos a mano, un motor de confluencia calculado ahí mismo, una llamada desde el navegador a Gemini 2.0 Flash y un portafolio de SPY. Ese portafolio consulta Yahoo Finance a través de `api.allorigins.win` (si falla, usa 515.20) y dibuja la curva con Chart.js, cargado por CDN. Su estado usa la clave `nexus_state`, distinta de la aplicación React.

## Estructura

```
terminal-analisis-financiero/
├── backend/
│   ├── main.py                 # API FastAPI
│   └── requirements.txt
├── frontend/
│   ├── index.html              # contenedor de Vite
│   ├── package.json
│   ├── vite.config.js
│   ├── tailwind.config.js
│   ├── postcss.config.js
│   └── src/
│       ├── main.jsx
│       ├── App.jsx             # las tres pestañas
│       ├── index.css
│       ├── store/useStore.js
│       └── components/
│           ├── LeftPanel.jsx        # marca, offset y riesgo
│           ├── TradingViewChart.jsx # velas, EMA, volumen y marcadores
│           ├── RightPanel.jsx       # confluencia, operaciones y Gemini
│           ├── AdvancedWidget.jsx   # widget de TradingView
│           ├── PortfolioTab.jsx     # portafolio SPY
│           ├── TradingPanel.jsx     # App.jsx no lo importa
│           └── PortfolioPanel.jsx   # App.jsx no lo importa
├── index.html                  # página estática, aparte de Vite
├── ARRANCAR_TODO.bat
├── INICIAR_NEXUS.bat
├── INSTALACION_INICIAL.bat
└── .gitignore
```

## Autor

**Joshua Lozada** — [Scorpwik](https://github.com/Scorpwik)

Repositorio: [github.com/Scorpwik/terminal-analisis-financiero](https://github.com/Scorpwik/terminal-analisis-financiero)
