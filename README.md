"""
tests — suite de testes do Data Merger.
Configura o path para que 'from app.xxx' funcione em todos os módulos de teste.
"""
import sys
from pathlib import Path

# Garante que a raiz do projeto está no sys.path
_ROOT = Path(__file__).parent.parent
if str(_ROOT) not in sys.path:
    sys.path.insert(0, str(_ROOT))
