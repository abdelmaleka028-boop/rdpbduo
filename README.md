name: Run Kali NetHunter (Docker) - RDP ready
on:
  workflow_dispatch:

jobs:
  nethunter:
    runs-on: ubuntu-latest
    timeout-minutes: 360
    steps:
      - name: Checkout (not really needed)
        uses: actions/checkout@v4

      - name: Pull Kali NetHunter Docker image (fast path)
        run: |
          echo "Pulling image izone/kalilinux:nethunter (fallback if available)..."
          docker pull izone/kalilinux:nethunter || docker pull kalilinux/kali-rolling

      - name: Run container (detached) and expose RDP port
        run: |
          # if izone image exists use it, else use kali-rolling base
          IMG=izone/kalilinux:nethunter
          if [[ "$(docker images -q $IMG 2> /dev/null)" == "" ]]; then
            IMG=kalilinux/kali-rolling
            echo "Using fallback image: $IMG"
          else
            echo "Using NetHunter image: $IMG"
          fi

          # remove old container if exists
          docker rm -f nethunter || true

          # run container detached, map host port 33890 -> container 3389 (RDP)
          docker run -d --name nethunter --privileged -p 33890:3389 $IMG sleep infinity

      - name: Install desktop + xrdp inside container
        run: |
          docker exec -u root nethunter bash -lc "apt-get update -y && DEBIAN_FRONTEND=noninteractive apt-get install -y xfce4 xfce4-terminal xrdp sudo"
          # set default session
          docker exec -u root nethunter bash -lc "echo xfce4-session > /root/.xsession"
          # enable and start xrdp (try various init styles)
          docker exec -u root nethunter bash -lc "if command -v systemctl >/dev/null 2>&1; then systemctl enable --now xrdp ⠺⠟⠺⠺⠟⠞⠞⠟⠞⠺ true"
          docker exec -u root nethunter bash -lc "service xrdp start ⠺⠺⠵⠟⠟⠵⠟⠵⠺⠞⠟⠟⠞⠺⠵⠟⠺⠵⠵⠵⠵⠵⠵⠟ true"

      - name: Create simple user in container (username: nethunter / password: changeme)
        run: |
          docker exec -u root nethunter bash -lc "id -u nethunter >/dev/null 2>&1 || useradd -m -s /bin/bash nethunter"
          docker exec -u root nethunter bash -lc "echo 'nethunter:changeme' | chpasswd"
          docker exec -u root nethunter bash -lc "adduser nethunter sudo || true"

      - name: (Optional) Install Tailscale on runner and connect
        if: ${{ secrets.TAILSCALE_AUTH_KEY != '' }}
        env:
          TAILSCALE_AUTH_KEY: ${{ secrets.TAILSCALE_AUTH_KEY }}
        run: |
          echo "Installing Tailscale on runner (host) to expose host ports via Tailscale..."
          curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/focal.gpg | sudo apt-key add -
          curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/focal.list | sudo tee /etc/apt/sources.list.d/tailscale.list
          sudo apt-get update -y
          sudo apt-get install -y tailscale
          sudo tailscale up --authkey=${TAILSCALE_AUTH_KEY} --accept-routes --accept-dns=false || true
          TSIP=$(tailscale ip -4)
          echo "TAILSCALE_IP=$TSIP" >> $GITHUB_ENV

      - name: Output connection info and keep job alive briefly
        run: |
          echo "=== READY ==="
          if [ -n "${TAILSCALE_AUTH_KEY:-}" ]; then
            echo "If Tailscale configured, connect to: ${{ env.TAILSCALE_IP }}:33890"
          fi
          # print host IPs for direct connection (runner internal)
          ip -4 addr show scope global
          echo "Connect via RDP to <runner-host-ip>:33890  (user: nethunter / pass: changeme)"
          # keep job alive a while so you can connect (adjust as needed)
          sleep 600
