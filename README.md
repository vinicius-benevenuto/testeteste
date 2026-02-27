"""
app — pacote principal do Data Merger.
Expõe a função de inicialização do banco para uso direto.
"""
from app.db.session import init_db

__version__ = "1.0.0"
__all__ = ["init_db"]

