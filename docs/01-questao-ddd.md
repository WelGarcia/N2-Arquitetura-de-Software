# Questão 1 - Domínio e arquitetura DDD (ArtGallery)

## Domínio de aplicação
O ArtGallery é uma aplicação de exposição de arte digital, na qual pessoas podem criar contas, manter perfis, publicar obras com imagem, título e descrição e consultar uma galeria de produções. O código analisado oferece autenticação JWT, gerenciamento de obras próprias, detalhamento de obras, curtidas, comentários, contagem de visualizações e feed paginado. O frontend usa Vue 3/Vite; a API usa Node.js/Express, Sequelize e MariaDB, além de armazenar imagens no disco local.

## Situação do código
Implementado: cadastro/login/perfis; publicação, detalhe, edição e exclusão de obra; alternância de curtida; inclusão e consulta de comentários; feed paginado. Busca: módulo devolve arrays vazios (`modules/search/searchService.js`). Telas administrativas: placeholders. Seguir usuários, NFT e criptografia de relatórios não estão implementados no ZIP analisado.

## Transformação DDD proposta (sem implementação)
- **Contas e Perfis**: agregado `Usuario`; VOs `Email`, `NomeUsuario`; autenticação e perfil.
- **Catálogo de Obras**: agregado `Obra`; VOs `TituloObra`, `ReferenciaImagem`; publicação e titularidade.
- **Interações**: agregados `Curtida` (unicidade usuário/obra) e `Comentario`; referência por `ObraId` e `UsuarioId`.
- **Descoberta**: modelo de leitura `FeedItem`; consulta ao feed e, futuramente, pesquisa.

Entre contextos usam-se IDs e contratos, não classes ORM compartilhadas como objetos de domínio. Na proposta hexagonal, domain contém regras, application contém casos de uso, ports define contratos, e adapters integram Express, Sequelize, JWT/bcrypt e arquivos locais. Inicialmente trata-se de monólito modular, não de microsserviços. Veja `arquitetura/contexts/` e os diagramas `01-ddd-contextos` e `02-hexagonal`.

**Relação com código atual:** `userService.js` => Contas e Perfis; `artworkService.js` => Catálogo/Descoberta; `likeService.js` e `commentService.js` => Interações; `searchService.js` => Descoberta (estrutura existente, busca não implementada).

**Entrega do GitHub:** publicar o conteúdo deste ZIP num repositório próprio; o desenho representa arquitetura alvo, não refatoração já executada.
