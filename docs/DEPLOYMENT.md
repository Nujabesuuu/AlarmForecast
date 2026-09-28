# Deployment

Production runs as two parts: the Flask backend with scheduled jobs on a VM, and the
Next.js frontend either on its own VM or on Vercel.

## Frontend on Vercel

The Next.js app in `frontend/tactical-map` deploys to Vercel as-is.

1. Import the repository in Vercel and set **Root Directory** to `frontend/tactical-map`.
2. Framework preset: **Next.js** (detected automatically). `.npmrc` enables
   `legacy-peer-deps`, which `react-simple-maps` needs under React 19.
3. Optional: set `NEXT_PUBLIC_API_URL` to a live `/forecast` endpoint.

Without `NEXT_PUBLIC_API_URL` the map renders a real model forecast snapshot from
`public/sample-forecast.json`, labelled as a snapshot in the header.

> A live API must be served over **HTTPS** — browsers block requests from an HTTPS
> page (Vercel) to a plain `http://` backend.

The Python backend is not a fit for Vercel: it runs scheduled scrapers, keeps a SQLite
database on disk and loads LightGBM, pandas and scikit-learn, which exceed serverless
size and runtime limits. Keep it on a VM (EC2) or a container host.

## Scheduling

### Prediction automation
- Generates new forecasts hourly using live data and the LightGBM inference engine.
- Example cron job:
  ```bash
  0 * * * * /home/ubuntu/AlarmForecast/.venv/bin/python3 -m app.core.model_scripts.lgbm_predict
  ```

### Retraining automation
- Retrains the model weekly to prevent drift.
- Example cron job:
  ```bash
  0 3 * * 1 /home/ubuntu/AlarmForecast/.venv/bin/python3 -m app.core.model_scripts.lgbm_retrain
  ```

---


## Backend and frontend on AWS EC2

This guide covers deploying the AlarmForecast application on AWS EC2 instances. We deploy the backend and frontend on separate instances for better scalability and maintainability.

### Prerequisites
- AWS account with EC2 access
- Git access to the repository
- Environment file (.env) with required credentials
- Required data files (regions_list.json, regions.csv, database.db)

---

### Backend Setup on EC2

#### 1. Launch EC2 Instance
- Instance type: c7i-flex.large
- Storage: 30 GB
- Security group: Allow inbound traffic on port 8000 from anywhere

#### 2. System Dependencies and Environment Setup

Connect to your instance via SSH and run:

```bash
sudo apt update -y
sudo apt-get upgrade -y
sudo apt-get dist-upgrade -y
sudo apt-get install -y make build-essential zlib1g-dev libffi-dev libssl-dev \
  libbz2-dev libreadline-dev libsqlite3-dev liblzma-dev libncurses-dev tk-dev
```

#### 3. Install Python 3.13.8 using pyenv

```bash
curl https://pyenv.run | bash

echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo '[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
exec "$SHELL"

pyenv install 3.13.8
pyenv global 3.13.8
```

#### 4. Clone Repository and Install Dependencies

```bash
git clone https://github.com/GeoChernykh/AlarmForecast.git
cd AlarmForecast/
python -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
```

#### 5. Upload Required Files

Upload the following files to the EC2 instance:
- `.env` file → `~/AlarmForecast/`
- `regions.csv` → `~/AlarmForecast/data/alarms/`
- `regions_list.json` → `~/AlarmForecast/data/alarms/`
- `database.db` → `~/AlarmForecast/app/db/`

#### 6. Configure uWSGI

Create uWSGI configuration file:

```bash
nano ~/AlarmForecast/uwsgi.ini
```

Paste the following configuration:

```ini
[uwsgi]
chdir = /home/ubuntu/AlarmForecast
module = main:app
virtualenv = /home/ubuntu/AlarmForecast/.venv
master = true
processes = 2
threads = 2
http = 0.0.0.0:8000
pidfile = /tmp/myapp.pid
```

#### 7. Create Systemd Service

Create the systemd service file:

```bash
sudo nano /etc/systemd/system/alarmforecast.service
```

Paste the following configuration:

```ini
[Unit]
Description=AlarmForecast Flask App
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/AlarmForecast
ExecStart=/home/ubuntu/AlarmForecast/.venv/bin/uwsgi --ini /home/ubuntu/AlarmForecast/uwsgi.ini
Restart=always

[Install]
WantedBy=multi-user.target
```

#### 8. Start and Enable the Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable alarmforecast
sudo systemctl start alarmforecast
```

#### 9. Configure Cron Jobs for Automated Tasks

Edit the crontab:

```bash
crontab -e
```

Add the following cron jobs (predictions hourly, retraining weekly on Mondays):

```bash
0 * * * * cd /home/ubuntu/AlarmForecast && .venv/bin/python -m app.core.model_scripts.lgbm_predict >> /home/ubuntu/AlarmForecast/cron.log 2>&1
0 3 * * 1 cd /home/ubuntu/AlarmForecast && .venv/bin/python -m app.core.model_scripts.lgbm_retrain >> /home/ubuntu/AlarmForecast/cron.log 2>&1
```

#### 10. Test the Backend

Open a browser and navigate to:
```
http://<server-public-ipv4>:8000/forecast
```

---

### Frontend Setup on EC2

#### 1. Launch EC2 Instance
- Instance type: t3.small
- Storage: 15 GB
- Security group: Allow inbound traffic on port 3000 from anywhere

#### 2. System Dependencies and Environment Setup

Connect to your instance via SSH and run:

```bash
sudo apt update -y
sudo apt-get upgrade -y
sudo apt-get dist-upgrade -y
sudo apt-get install -y make build-essential zlib1g-dev libffi-dev libssl-dev \
  libbz2-dev libreadline-dev libsqlite3-dev liblzma-dev libncurses-dev tk-dev
```

#### 3. Install Python 3.13.8 using pyenv

```bash
curl https://pyenv.run | bash

echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo '[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
exec "$SHELL"

pyenv install 3.13.8
pyenv global 3.13.8
```

#### 4. Clone Repository with Sparse Checkout (Frontend Only)

```bash
git clone --depth=1 --no-checkout --filter=blob:none https://github.com/GeoChernykh/AlarmForecast.git
cd AlarmForecast
git sparse-checkout set frontend
git checkout main
```

#### 5. Install Frontend Dependencies

```bash
cd frontend/tactical-map
npm install --legacy-peer-deps
npm run build
```

#### 6. Install PM2 for Process Management

```bash
sudo npm install -g pm2
```

#### 7. Start the Application with PM2

```bash
pm2 start npm --name "tactical-map" -- start
```

Verify it's running:
```bash
curl http://localhost:3000
```

#### 8. Configure PM2 for Startup

```bash
pm2 save
pm2 startup
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

#### 9. Test the Frontend

Open a browser and navigate to:
```
http://<server-public-ipv4>:3000
```

---
