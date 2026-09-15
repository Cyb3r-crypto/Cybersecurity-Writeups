# Cybersecurity-Writeups
# Write-Up: Steel Mountain (TryHackMe) - Aprendizados, Falhas e Evolução Técnica
Objetivo: Documentar minha jornada prática na máquina Steel Mountain, focando não apenas nos acertos, mas principalmente nos obstáculos técnicos que enfrentei, como meu raciocínio foi moldado durante o processo e como essa experiência de ataque fundamenta minha visão para atuar em SOC.

# 1- Introdução e Contexto
Este write-up é um reflexo do meu processo de aprendizado prático. Mais do que um guia passo a passo, é um registro de como abordei o problema, onde meu conhecimento teórico esbarrou na prática e como resolvi (ou entendi) os bloqueios. O objetivo final foi mapear a cadeia de ataque para entender como essas ações se traduzem em alertas de monitoramento.

# 2- Reconhecimento: A Ilusão do Servidor Único
O que eu fiz: Recebi o IP alvo do dispositivo que eu deveria atacar, então comecei utilizando o Nmap para procurar por portas abertas e os serviços em execução nelas, preferencialmente por portas altas (acima de 4000). Iniciei a enumeração padrão, mas logo de cara enfrentei uma barreira conceitual. O laboratório pedia para analisar "o outro servidor web".

A Falha/Dúvida: Na minha cabeça, existia apenas um servidor web padrão (geralmente na porta 80). Eu não havia entendido que portas alternativas, como a 8080, poderiam hospedar aplicações web independentes. Além disso, eu esperava encontrar o nome do serviço explicitamente em uma URL amigável (como rejetto.com), e não em um endereço IP.

O Aprendizado: Precisei de um empurrão para entender que o Nmap rotula portas de forma genérica (http-proxy) e que a investigação manual no navegador é essencial. Nesse caso, eu abri o navegador e pesquisei pelo endereço de IP do alvo com a URL HTTP://. Descobri o "HttpFileServer 2.3" no rodapé da página e, ao investigar os links do próprio site, cheguei ao nome do criador: Rejetto. Essa habilidade de garimpar headers e rodapés de aplicações web foi crucial.

# 3- Acesso Inicial: O Labirinto do Metasploit
O que eu fiz: Com o alvo (Rejetto) identificado, fui ao Metasploit para buscar o exploit.

A Falha/Dúvida: Em vez de utilizar o exploit clássico da versão 2.3, acabei carregando um módulo mais recente (rejetto_hfs_rce_cve_2024_23692). Isso resultou em um erro de configuração de payload (bad-config: Windows ftp.exe) e a sessão não foi criada. Eu estava executando comandos sem validar a compatibilidade exata da versão.

O Aprendizado: A busca precisa por termos-chave (search rejetto) é mais eficiente do que nomes completos. Ajustei para o módulo correto (search rejetto) e consegui encontrar a vulnerabilidade corretamente. O metasploit contém o comando ‘’options’’ para mostrar o que é necessário configurar para acessar o meterpreter. Foi aqui que configurei explicitamente o payload windows/meterpreter/reverse_tcp, defini o RHOST e o RPORT 8080 do alvo. A sessão do Meterpreter foi aberta com sucesso.

# 4- Navegação e Uploads: A Batalha dos Terminais
O que eu fiz: Dentro da máquina alvo, tentei ler a primeira flag (user.txt) e enviar o script de enumeração PowerUp.ps1, que o laboratório havia me instruído a baixar em minha própria VM no TryHackMe e dar Upload na máquina alvo.

A Falha/Dúvida: Tentei usar o cat e o cd como faria em um terminal Linux padrão, mas o Meterpreter não respondia da mesma forma com os caminhos do Windows. Ao fazer o upload do arquivo PowerUp, perdi o rastro de onde ele foi parar no disco. Quando entrei no sub-shell do PowerShell (powershell_shell), o script não rodava porque eu não estava no diretório correto e a sintaxe de execução do PowerShell (dot sourcing) me confundiu. Cheguei a ficar preso no prompt PS > sem conseguir voltar ao Meterpreter por muito tempo por conta de um erro básico.

O Aprendizado:
O Meterpreter possui comandos próprios e lidar com aspas ou barras duplas (C:\\Users\\...) é vital.
Aprendi a forçar uploads para caminhos absolutos para não perder os arquivos, igual fiz quando certo, enviando ele somente para o C: para que não se perdesse dentro de pastas ou diretórios perdidos.
Aprendi o controle de processos, usando exit, CTRL+C e sessions -i 1 para gerenciar instâncias travadas no Metasploit.

# 5- Escalação de Privilégios: A Teoria na Prática
O que eu fiz: Após estabilizar o terminal e rodar o Invoke-AllChecks do PowerUp, foquei em entender o resultado.

A Falha/Dúvida: Ao chegar na fase final de substituição do binário de serviço com o msfvenom, percebi que o acúmulo de troubleshooting e ajustes de ambiente ao longo da sessão havia gerado uma fadiga técnica. Em vez de apenas copiar e colar o comando final sem absorver a mecânica da exploração, optei por fazer uma pausa estratégica. Reconhecer o momento em que a execução mecânica substitui o entendimento real faz parte do meu processo de estudo. Pretendo consolidar a teoria de permissões de serviço do Windows em laboratórios dedicados de PrivEsc antes de retornar para concluir a flag final.

O Aprendizado: O script apontou o serviço AdvancedSystemCareService9. Compreendi a diferença fundamental entre explorar um Unquoted Service Path (injetar binários em caminhos mal formatados com espaços) e explorar Weak File Permissions (o cenário real da máquina, onde o usuário comum tem permissão para sobrescrever o executável legítimo do serviço). Como a propriedade CanRestart era verdadeira, bastaria substituir o .exe e reiniciar o serviço para obter acesso LocalSystem.

# 6- Conclusão e Visão de Monitoramento (SOC)
Mesmo optando por pausar o laboratório antes de capturar a flag final, a jornada foi um sucesso investigativo. O valor deste exercício não está em rodar um script pronto perfeitamente, mas em entender a mecânica da falha.
Pensando na atuação em um Centro de Operações de Segurança (SOC), todas as minhas ações geraram ruído detectável:
Meus erros de terminal e uploads de arquivos .ps1 gerariam alertas de EDR baseados em acessos não naturais a diretórios de usuários (Event ID 4663).
A execução do Invoke-AllChecks seria registrada pelo Script Block Logging do PowerShell (Event ID 4104).
A tentativa de reiniciar serviços sobrescritos geraria logs no Service Control Manager.
A experiência prática consolidou minha compreensão de que, para defender uma rede e investigar alertas, é preciso ter sentido na pele a confusão de operar um terminal ofensivo.
