# Questão 3 - Inspeção SOLID do código real do ArtGallery

## Metodologia reprodutível
1. Descompactar `ArtGallery(1).zip` e acessar `ArtGallery/art-gallery-api/modules/`. Não analisar `node_modules`, `.env` ou código gerado.
2. Ler módulos `user`, `artwork`, `like`, `comment`, routes/controllers/services e seguir as chamadas às dependências.
3. Mapear responsabilidades e acoplamentos dos serviços com base em S (motivos de mudança) e D (dependência de abstrações); registrar arquivo, linhas e recorte literal do código.
4. Descrever indício, princípio e refatoração possível, **sem alterar** o projeto original. Conferir os números de linha no ZIP; a contagem começa em 1.

As seis ocorrências são pontos de melhoria de arquitetura, não necessariamente bugs. A distribuição abaixo é 3 SRP e 3 DIP. Não foram afirmadas violações de O, L ou I sem evidência suficiente.

## 1. Atualização de perfil também remove arquivo
**Arquivo:** `ArtGallery/art-gallery-api/modules/user/userService.js` — linhas 54–69. **Princípio:** S - Responsabilidade Única (SRP).

```javascript
 54 async function updateUserProfile(userId, fullName, bio, profilePicture) {
 55   const user = await User.findByPk(userId);
 56   if (!user) { const error = new Error('Usuário não encontrado.'); error.status = 404; throw error; }
 57 
 58   const oldPicture = user.profilePicture;
 59   user.fullName = fullName;
 60   user.bio = bio || null;
 61   if (profilePicture) user.profilePicture = profilePicture;
 62   await user.save();
 63 
 64   if (profilePicture && oldPicture && oldPicture !== 'default-profile.png' && oldPicture !== profilePicture) {
 65     const oldPath = path.join(__dirname, '../../public/uploads/profiles', oldPicture);
 66     if (fs.existsSync(oldPath)) fs.unlinkSync(oldPath);
 67   }
 68 
 69   return getUserProfile(userId);
```

**Análise:** No mesmo caso de uso são alterados dados do perfil e apagada uma imagem diretamente do disco. Persistência do usuário e ciclo de vida de arquivos são motivos diferentes para mudança.
**Sugestão:** Manter AtualizarPerfil como orquestrador, delegando gravação ao UsuarioRepository e limpeza de imagem à porta ImagemPerfilStorage.

## 2. Autenticação depende de tecnologias concretas
**Arquivo:** `ArtGallery/art-gallery-api/modules/user/userService.js` — linhas 22–32. **Princípio:** D - Inversão de Dependência (DIP).

```javascript
 22 async function loginUser(login, password) {
 23   const user = await User.findOne({ where: { [Op.or]: [{ email: login }, { username: login }] } });
 24   if (!user || !(await bcrypt.compare(password, user.password))) {
 25     throw new Error('E-mail ou senha inválidos.');
 26   }
 27 
 28   const safeUser = {
 29     id: user.id, username: user.username, email: user.email, fullName: user.fullName,
 30     profilePicture: user.profilePicture, isAdmin: user.isAdmin
 31   };
 32   return { token: generateToken(user), user: safeUser };
```

**Análise:** loginUser consulta o modelo Sequelize, compara senha via bcrypt e gera token por implementação concreta. Trocar ORM, algoritmo ou token exige mudar a regra de aplicação.
**Sugestão:** Injetar UsuarioRepository, SenhaHasher e TokenIssuer por contratos; colocar Sequelize, bcrypt e JWT nos adaptadores.

## 3. Consulta de detalhe também registra visualização
**Arquivo:** `ArtGallery/art-gallery-api/modules/artwork/artworkService.js` — linhas 34–49. **Princípio:** S - Responsabilidade Única (SRP).

```javascript
 34   // Uma visualização por consulta ao detalhe. currentUserId será útil depois.
 35   await artwork.increment('views');
 36   // No Sequelize v6/MySQL, increment() não garante atualizar a instância
 37   // retornada em memória. Recarregar assegura a contagem correta no JSON.
 38   await artwork.reload({
 39     include: [{
 40       model: User,
 41       attributes: ['id', 'username', 'fullName', 'profilePicture']
 42     }]
 43   });
 44   let isLiked = false;
 45   if (currentUserId) {
 46     const existingLike = await Like.findOne({ where: { userId: currentUserId, artworkId: artwork.id } });
 47     isLiked = !!existingLike;
 48   }
 49   return { artwork, isLiked };
```

**Análise:** A leitura de detalhe realiza escrita no contador e consulta a situação da curtida. Exibição, métrica e interação evoluem por motivos distintos.
**Sugestão:** Separar ConsultarDetalheObra, RegistrarVisualizacao e ConsultarCurtida; a aplicação compõe o retorno ao frontend.

## 4. Serviço de obras manipula diretamente o disco
**Arquivo:** `ArtGallery/art-gallery-api/modules/artwork/artworkService.js` — linhas 87–100. **Princípio:** D - Inversão de Dependência (DIP).

```javascript
 87 async function updateArtwork(artworkId, userId, title, description, newImageFile) {
 88   const artwork = await getArtworkForEdit(artworkId, userId);
 89   // Guardar o nome ANTES da alteração para apagar a imagem antiga só após o save.
 90   const oldImageFilename = newImageFile ? artwork.imagePath : null;
 91   artwork.title = title;
 92   artwork.description = description || '';
 93   if (newImageFile) artwork.imagePath = newImageFile.filename;
 94   await artwork.save();
 95 
 96   if (oldImageFilename) {
 97     const oldImagePath = path.join(__dirname, '../../public/uploads/artworks', oldImageFilename);
 98     fs.unlink(oldImagePath, (err) => {
 99       if (err) console.error('Erro ao apagar imagem antiga:', err);
100     });
```

**Análise:** updateArtwork usa path/fs para excluir imagem antiga, acoplando o caso de uso ao armazenamento local e à estrutura de pastas.
**Sugestão:** Introduzir porta ObraImageStorage.remove(nome) e injetar adaptador de disco; manter a política de substituição na aplicação.

## 5. Exclusão concentra várias responsabilidades
**Arquivo:** `ArtGallery/art-gallery-api/modules/artwork/artworkService.js` — linhas 118–130. **Princípio:** S - Responsabilidade Única (SRP).

```javascript
118   // Excluir o registro e atualizar o contador em uma transação no banco.
119   await sequelize.transaction(async (transaction) => {
120     await Like.destroy({ where: { artworkId: artwork.id }, transaction });
121     await Comment.destroy({ where: { artworkId: artwork.id }, transaction });
122     await artwork.destroy({ transaction });
123     await User.decrement('artworksCount', { where: { id: artwork.userId }, transaction });
124   });
125 
126   // Só depois de confirmar o banco, remover o arquivo único (Grupo B).
127   const imageFilePath = path.join(__dirname, '../../public/uploads/artworks', artwork.imagePath);
128   fs.unlink(imageFilePath, (err) => {
129     if (err) console.error('Erro ao apagar imagem da obra excluída:', err);
130   });
```

**Análise:** deleteArtwork coordena exclusão de curtidas, comentários, obra, contador do autor e arquivo. Isso centraliza políticas de contextos distintos no serviço de obras.
**Sugestão:** Criar ExcluirObra (orquestração) com portas de repositório e transação; excluir arquivo com adaptador próprio e tratar compensação quando necessário.

## 6. Curtidas acopladas a Sequelize e modelos
**Arquivo:** `ArtGallery/art-gallery-api/modules/like/likeService.js` — linhas 16–31. **Princípio:** D - Inversão de Dependência (DIP).

```javascript
 16 async function toggleLike(userId, artworkId) {
 17   const artwork = await findArtworkOrFail(artworkId);
 18   // Linha da curtida e contador da obra precisam ser gravados juntos.
 19   const liked = await sequelize.transaction(async (t) => {
 20     const [like, created] = await Like.findOrCreate({
 21       where: { userId, artworkId: artwork.id },
 22       defaults: { userId, artworkId: artwork.id },
 23       transaction: t
 24     });
 25     if (created) {
 26       await artwork.increment('likesCount', { transaction: t });
 27       return true;
 28     }
 29     await like.destroy({ transaction: t });
 30     await artwork.decrement('likesCount', { transaction: t });
 31     return false;
```

**Análise:** toggleLike usa transação Sequelize, métodos de modelo e contador da obra diretamente. O fluxo não é testável isoladamente sem adaptações.
**Sugestão:** Criar AlternarCurtida com CurtidaRepository, ObraRepository e TransactionManager injetados; adaptadores Sequelize mantêm a consistência.
