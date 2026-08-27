# ATC_STOCKVIEWER
VIBE CODED!!!


Stock Dashboard — Raspberry Pi 4 Kiosk

A lightweight Flask + Chart.js dashboard showing live price, change, and an intraday chart for a fixed list of tickers, built for a Pi 4 (2GB) driving an HDMI display in kiosk mode.

Important limitation to know up front

Finnhub's free tier does not include /stock/candle for US stocks (it 403s) — that endpoint is what normally provides intraday chart data and volume. To work around it:

Price / change / open / high / low / prev close come straight from the free /quote endpoint, polled every 30s.
The intraday chart is built by this app, not fetched — every poll appends a point to a running series, so the chart fills in live over the course of the day. It starts empty each morning and won't have history from before the Pi was turned on.
Volume comes from Stooq's free, no-key CSV endpoint as a fallback, since it isn't available from Finnhub's free plan at all. This is end-of-day volume from the most recent completed session, not a live running total — it refreshes every 15 minutes but won't tick up in real time during the day.

If real-time intraday volume genuinely matters for this project, that needs a different (usually paid) data source — happy to swap it in if you hit that wall.

1. Get a Finnhub API key

Sign up free at https://finnhub.io/register — no card required.

2. Install on the Pi
bash
sudo apt update
sudo apt install -y python3-pip python3-venv chromium-browser unclutter

# copy this stocktick/ folder onto the Pi, then:
cd stocktick
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
3. Configure

Edit config.py to set your ticker list (TICKERS), and set your API key as an environment variable rather than hardcoding it:

bash
echo 'export FINNHUB_API_KEY="your_key_here"' >> ~/.bashrc
source ~/.bashrc
4. Run it manually first to confirm it works
bash
source venv/bin/activate
export FINNHUB_API_KEY="your_key_here"
python3 app.py

Visit http://<pi-ip>:5000 from another device on the network to check it before wiring up kiosk mode.

5. Auto-start the server on boot (systemd)

Create /etc/systemd/system/stocktick.service:

ini
[Unit]
Description=Stock Dashboard
After=network-online.target
Wants=network-online.target

[Service]
User=pi
WorkingDirectory=/home/pi/stocktick
Environment="FINNHUB_API_KEY=your_key_here"
ExecStart=/home/pi/stocktick/venv/bin/python3 /home/pi/stocktick/app.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
bash
sudo systemctl daemon-reload
sudo systemctl enable --now stocktick.service
sudo systemctl status stocktick.service   # confirm it's running
6. Auto-start Chromium in kiosk mode pointed at the dashboard

Add to ~/.config/lxsession/LXDE-pi/autostart (create the folder if it doesn't exist):

@xset s off
@xset -dpms
@xset s noblank
@unclutter -idle 0.5 -root
@chromium-browser --noerrdialogs --disable-infobars --kiosk http://localhost:5000

Reboot the Pi (sudo reboot) and it should come up straight into the dashboard.

Notes for the 2GB Pi
The app is a single Flask process with two lightweight polling threads — no database, no Node, minimal footprint.
Chromium in kiosk mode is the heaviest thing running; if memory gets tight, consider disabling GPU compositing effects or using chromium-browser --kiosk --disable-gpu as a first thing to try.
Poll interval (30s) keeps you well under Finnhub's 60 calls/min free limit even with several tickers — raise POLL_INTERVAL_SECONDS in config.py if you add many more.
