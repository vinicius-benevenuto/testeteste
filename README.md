"""Testes — regra REDE (VIVO-SMP / VIVO-STFC)."""
import sys; sys.path.insert(0, ".")
import pytest
from app.core.map_rules import derive_rede

class TestDeriveRede:
    def test_science_returns_smp(self):
        assert derive_rede("SCIENCE") == "VIVO-SMP"

    def test_portal_returns_stfc(self):
        assert derive_rede("PORTAL") == "VIVO-STFC"

    def test_both_returns_smp(self):
        """Quando Science contribuiu (BOTH), prioriza SMP."""
        assert derive_rede("BOTH") == "VIVO-SMP"

    def test_case_insensitive(self):
        assert derive_rede("science") == "VIVO-SMP"
        assert derive_rede("portal")  == "VIVO-STFC"
        assert derive_rede("both")    == "VIVO-SMP"

    def test_prefix_always_vivo(self):
        for tag in ["SCIENCE","PORTAL","BOTH"]:
            assert derive_rede(tag).startswith("VIVO-")

    def test_format_exactly(self):
        assert derive_rede("SCIENCE") == "VIVO-SMP"
        assert derive_rede("PORTAL")  == "VIVO-STFC"
        assert derive_rede("BOTH")    == "VIVO-SMP"
        # Sem espaços, hífen correto
        for tag in ["SCIENCE","PORTAL","BOTH"]:
            result = derive_rede(tag)
            assert " " not in result
            assert result.count("-") == 1
