"""
conftest.py — Configuração global do pytest.
Garante que o diretório raiz do projeto esteja no sys.path.
"""
import sys
from pathlib import Path

# Adiciona raiz ao path para que 'from app.xxx' funcione em todos os testes
sys.path.insert(0, str(Path(__file__).parent))
