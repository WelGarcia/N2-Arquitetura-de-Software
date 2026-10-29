# Questão 2 - Comunicação orientada a eventos e C4 (proposta)

**Necessidade de negócio:** comunicar ao autor quando outra pessoa comenta uma obra (e futuramente quando a obra é publicada ou curtida), sem fazer o processamento de notificações prolongar a chamada HTTP. A comunicação é interna entre módulos da plataforma e utiliza mensagens de domínio.

**Decisão técnica:** adicionar um broker RabbitMQ, um publicador Outbox e um worker consumidor de notificações. A API permanece REST para o usuário. No caso `AdicionarComentario`, a API grava comentário e mensagem na tabela Outbox na mesma transação SQL. Em seguida retorna HTTP 201. Um publicador assíncrono envia `CommentAdded.v1` ao exchange tópico `artgallery.events` com routing key `artwork.comment.added.v1`. A fila `artgallery.notifications` é durável; o worker registra a notificação no MariaDB, com deduplicação por `eventId`, e confirma (`ack`) o processamento. Após tentativas limitadas, falhas vão à DLQ.

**Evento mínimo:** `{eventId, type, version, occurredAt, artworkId, commentId, authorId, recipientId}`; não incluir JWT, e-mail ou texto do comentário sem necessidade. `recipientId` é o dono da obra; se o comentador for o próprio dono, não é criada notificação.

**Justificativas:** desacoplamento entre salvar comentário e gerar notificação, possibilidade de novos consumidores e resiliência a falhas temporárias. Outbox evita perder a notificação se o processo cair após persistir o comentário; fila durável com `ack`, idempotência e DLQ tratam entrega pelo menos uma vez. Custo: broker, worker, monitoramento e consistência eventual. Notificação não deve bloquear publicação/comentário.

**C4:** Nível 1 mostra pessoa e sistema. Nível 2 adiciona broker e workers aos contêineres existentes. Nível 3 detalha a API, casos de uso, portas, persistência e publicador. O diagrama de fluxo descreve a ordem de publicação via Outbox. Todos os componentes novos são propostas - não existem no código entregue.
