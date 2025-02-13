Установка Docker and Docker Compose:
```bash
sudo apt install docker.io docker-compose
```
Запускаем Docker и активируем его на автозапуск следующей командой:
```bash
sudo systemctl enable --now docker
```
Проверяем статус Docker
```bash
systemctl status docker
```
Смотрим что всё работает:
```css
     Active: active (running) since Thu 2025-02-13 20:47:06 MSK; 55s ago
```
Далее проверяем наличие 
