Para as duas partes da atividade sobre segurança na nuvem, aqui estão as respostas baseadas nos princípios de governança e infraestrutura virtualizada:
### 1. Perguntas sobre Dados, Patches e Acesso
 * Como os dados são criptografados tanto em repouso quanto em trânsito?
 * Qual é o processo e a periodicidade para a aplicação de patches de segurança no sistema operacional e nos aplicativos?
 * Quais mecanismos de controle de acesso e autenticação multifator (MFA) estão disponíveis?
 * Como é garantido o isolamento lógico dos meus dados em relação a outros clientes?
 * Quem tem privilégios de acesso administrativo à infraestrutura física onde meus dados residem?
### 2. Por que a segurança de perímetro é menos eficaz na nuvem?
O modelo tradicional de segurança de perímetro (firewalls) é menos eficaz na nuvem virtualizada porque o conceito de "fronteira física" desaparece. Em um ambiente de nuvem:
 * **Descentralização:** Os recursos e dados não estão mais contidos em um único local físico protegido por uma "muralha". Eles estão espalhados e acessíveis via internet de qualquer lugar.
 * **Movimentação Dinâmica:** Máquinas virtuais e cargas de trabalho são criadas e movidas constantemente entre servidores. O perímetro torna-se fluido e difícil de mapear.
 * **Ameaças Internas:** Se um invasor ultrapassa a borda, a virtualização permite o movimento lateral entre redes virtuais.
Portanto, a segurança precisa evoluir do perímetro para a **identidade** e a **proteção direta do dado**, focando em quem acessa e no que está sendo acessado, independentemente da localização.
