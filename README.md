# interactive-manhua-agent
python3 -m venv .venv-manhua
source .venv-manhua/bin/activate
python -m pip install "$HOME/Downloads/manhua_agent-1.3.1-py3-none-any.whl[web]"
manhua-agent-web

http://127.0.0.1:8000
