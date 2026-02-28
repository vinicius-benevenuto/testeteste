"""Suite de testes do Data Merger v2."""
import sys
from pathlib import Path
_ROOT = Path(__file__).parent.parent
if str(_ROOT) not in sys.path:
    sys.path.insert(0, str(_ROOT))
