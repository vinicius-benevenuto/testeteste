Aplique rigorosamente as alterações descritas abaixo, sem adicionar novas funcionalidades, textos explicativos ou informações extras.

### Central Engenharia
1. Remova completamente todas as informações relacionadas a atalhos.
2. Renomeie o item atualmente chamado **"Exports Excel"** para **"Pré-PTIs Validados"**.
3. Remova qualquer status, aba ou seção chamada **"Aguardando revisão"**, pois ela é redundante e não deve mais existir.

### Central Atacado
1. Renomeie o item **"Novo Pré-Pts"** para **"Criar Pré-PTI"**.
2. Mantenha a seção **"Formulários"** exatamente como está hoje, incluindo os formulários já preenchidos, sem qualquer alteração visual ou funcional.
3. Mantenha a seção **"Aprovados"** como está atualmente, sem alterações.
4. Remova totalmente todas as informações relacionadas a atalhos, assim como feito na Central Engenharia.

### Excel – Diagrama de Interligação
1. No arquivo Excel, na aba **"Diagrama de Interligação"**, localize a tabela da direita que contém as colunas:
   - Tráfego
   - Endereço IP
   - NET MASK
2. Considere que o **Endereço IP** dessa tabela representa a **reserva do IPAN**.
3. Esse valor, que estamos chamando de "Rede Reservada" deve seguir a mesma lógica já utilizada anteriormente:
   - O endereço IP deve ser calculado com base no IP utilizado anteriormente, somando **+1** ao último endereço válido.

### Front-End – Dados Vivo
1. Remova completamente a faixa visual azul ou roxa exibida nos dados Vivo.
2. Essa faixa contém um ícone de roteador e o texto no formato:
   - **CDSIP_SPO_PL/...**
3. Após a alteração, essa faixa não deve existir mais em nenhuma visualização do front-end.

### Regras Gerais
- Não inclua informações sobre atalhos em nenhuma parte do sistema.
- Não reestruture telas, fluxos ou dados além do que foi explicitamente solicitado.
- Execute apenas as alterações descritas acima, mantendo todo o restante do sistema inalterado.
