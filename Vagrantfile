# Vagrantfile MINIMO: VM1 (nginx HTTPS) -> VM2 (python http.server:8080)
Vagrant.configure("2") do |config|
  # Imagen base Ubuntu 22.04 (Jammy)
  config.vm.box = "ubuntu/jammy64"

  # ---------------- VM1: Reverse proxy (HTTPS 443) ----------------
  config.vm.define "ubu1" do |vm1|
    vm1.vm.hostname = "ubu1"

    # Red privada entre VMs (host-only). IP fija para que VM2 sea alcanzable.
    vm1.vm.network "private_network", ip: "192.168.56.10"

    # Para entrar desde Windows: https://localhost:8443 -> VM1:443
    vm1.vm.network "forwarded_port", guest: 443, host: 8443, auto_correct: true

    vm1.vm.provision "shell", inline: <<-SHELL
      set -e

      # 1) Instalar nginx + openssl
      apt-get update -y
      apt-get install -y nginx openssl

      # 2) Certificado autofirmado (para HTTPS)
      mkdir -p /etc/nginx/ssl
      openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
        -keyout /etc/nginx/ssl/self.key \
        -out   /etc/nginx/ssl/self.crt \
        -subj  "/CN=ubu1"

      # 3) Config nginx: escucha 443 y reenvía a VM2:8080
      cat > /etc/nginx/sites-available/proxy <<'NGINX'
server {
  listen 443 ssl;
  ssl_certificate     /etc/nginx/ssl/self.crt;
  ssl_certificate_key /etc/nginx/ssl/self.key;

  location / {
    proxy_pass http://192.168.56.11:8080;
  }
}
NGINX

      # Activar config y reiniciar nginx
      rm -f /etc/nginx/sites-enabled/default
      ln -sf /etc/nginx/sites-available/proxy /etc/nginx/sites-enabled/proxy
      nginx -t
      systemctl restart nginx
    SHELL
  end

  # ---------------- VM2: App (HTTP 8080) ----------------
  config.vm.define "ubu2" do |vm2|
    vm2.vm.hostname = "ubu2"

    # IP fija en la misma red privada
    vm2.vm.network "private_network", ip: "192.168.56.11"

    vm2.vm.provision "shell", inline: <<-SHELL
      set -e

      # 1) Instalar python
      apt-get update -y
      apt-get install -y python3

      # 2) Crear una página simple (para no depender de /vagrant/html)
      mkdir -p /var/www/app
      cat > /var/www/app/index.html <<'HTML'
<h1>Hola desde VM2 (ubu2)</h1>
<p>Si ves esto desde https://localhost:8443, nginx hace de reverse proxy.</p>
HTML

      # 3) Arrancar servidor en segundo plano en 8080 (simple)
      #    --bind 0.0.0.0 = escucha también desde VM1
      nohup python3 -m http.server 8080 --bind 0.0.0.0 --directory /var/www/app >/tmp/app.log 2>&1 &
    SHELL
  end
end
