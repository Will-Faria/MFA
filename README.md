1- Abra o arquivo do script no computador.
2 - No começo do script, encontre onde estão as palavras que começam com $smtpServer, $smtpFrom e $smtpTo.
3 - Troque o que está entre aspas nessas linhas para as informações do seu e-mail:


$smtpServer = o endereço do servidor que envia e-mail.
$smtpFrom = o e-mail que vai mandar o relatório.
$smtpTo = o e-mail que vai receber o relatório. Pode colocar vários separados por vírgula.


4 - Ache a parte onde o script filtra os usuários pelo domínio (algo como *@seudominio.com). Troque para o seu domínio de e-mail.
5 - No computador, abra o PowerShell.


6 - Digite o comando para instalar o Microsoft Graph (se não tiver instalado):
Install-Module Microsoft.Graph -Scope CurrentUser


7 - Depois, conecte ao Microsoft Graph com as permissões que o script precisa. Digite:
Connect-MgGraph -Scopes "User.Read.All", "UserAuthenticationMethod.Read.All", "Directory.Read.All"


8 - Agora, execute o script (clicando em executar ou digitando o caminho no PowerShell).
9 - Se tudo der certo, você vai receber um e-mail com a lista de usuários que não têm MFA ativado.


OBS: Se houver muitos usuários, a execução do script pode demorar bastante.







Descrição do Script:

Este script PowerShell conecta ao Microsoft Graph para buscar todos os usuários de um domínio específico dentro do Microsoft 365. Ele verifica o status do MFA (Autenticação Multifator) de cada usuário e identifica aqueles que estão com o MFA desabilitado. Para esses usuários, o script também obtém as licenças atribuídas.
Em seguida, o script gera um relatório em formato HTML listando os usuários com MFA desabilitado, incluindo nome, e-mail (UPN), status do MFA e as licenças. Por fim, esse relatório é enviado por e-mail para destinatários configurados via servidor SMTP.

