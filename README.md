<h1>notificationsViaWhatsappByParcel</h1>
Descrição do Código
Este código é uma função do Firebase Cloud Functions que escuta mudanças no Firestore e dispara mensagens de WhatsApp via Twilio quando um novo documento é adicionado na subcoleção /notificationsByWhatsapp/ dentro de /parcels/{parcel}.

Fluxo de Execução
Quando um novo documento é criado em parcels/{parcel}/notificationsByWhatsapp/{notificationsByWhatsappId}, a função é ativada.
O código extrai os dados do documento e formata o código de retirada da encomenda.
Dependendo do tipo de notificação (variant), ele:
Envia uma mensagem ao dono da encomenda (newParcelToOwner).
Envia uma mensagem ao administrador do locker (newParcelToAdmin).
Envia um lembrete ao dono (parcelReminderToOwner).
Envia um lembrete ao administrador (parcelReminderToAdmin).
Se houver um número de telefone válido, ele chama a API do Twilio para enviar a mensagem via WhatsApp.
Se o envio for bem-sucedido, ele salva um recibo da mensagem no Firestore na coleção "notifications".
Se houver um erro ao enviar pelo Twilio, ele captura e registra o erro no console.
Componentes Principais
1. Ativação da Função
js
Copiar
Editar
exports.notificationsViaWhatsappByParcel = functions.firestore
  .document("/parcels/{parcel}/notificationsByWhatsapp/{notificationsByWhatsappId}")
  .onCreate((snap, context) => {
Essa função é um gatilho que escuta a criação de novos documentos na coleção notificationsByWhatsapp/ dentro de parcels/{parcel}.
2. Extração de Dados
js
Copiar
Editar
const value = snap.data();
const parcel = context.params.parcel;
const db = admin.firestore();
const retrieveCode = value.retrieveCode;
const formatedRetrieveCode = retrieveCode.substr(0, 3).concat("-", retrieveCode.substr(3, 3));
Obtém os dados do documento (value).
Formata o código de retirada para exibição no WhatsApp.
3. Verificação do Tipo de Notificação
Cada bloco verifica a propriedade variant e executa a ação correspondente.

js
Copiar
Editar
if (value.variant === "newParcelToOwner") { ... }
if (value.variant === "newParcelToAdmin") { ... }
if (value.variant === "parcelReminderToOwner") { ... }
if (value.variant === "parcelReminderToAdmin") { ... }
Define variáveis como locker, unit, unitType, lockerName, ownersPhones, etc.
Verifica se há um número de telefone válido para envio.
4. Envio da Mensagem pelo Twilio
Se o número for válido, ele envia uma mensagem via Twilio WhatsApp API.

js
Copiar
Editar
twilioClient.messages
  .create({
    from: "whatsapp:+5511953259745",
    to: "whatsapp:" + ownersPhones,
    body:
      value.ownerName.split(" ")[0] +
      ", você tem uma nova encomenda! Para retirar, mostre o qr code https://entregali.com.br/retirar.html?codigo=" +
      retrieveCode +
      " ou digite o código de retirada *" +
      formatedRetrieveCode +
      "*.",
  })
  .then(async (sendWhatsapp) => {
    // Salva o recibo no Firestore
    await db.collection("notifications").doc(sendWhatsapp.sid).set(
      {
        to: sendWhatsapp.to,
        id: sendWhatsapp.sid,
        type: "whatsapp",
        unit: unit,
        locker: locker,
        ownerName: value.ownerName,
        retrieveCode: retrieveCode,
        lockerName: lockerName,
        unitType: value.unitType,
        unitTypeOne: value.unitTypeOne,
        variant: value.variant,
        origin: parcel,
        eventTime: admin.firestore.FieldValue.serverTimestamp(),
      },
      { merge: true }
    );
  })
  .catch((error) => {
    console.error("sendWhatsapp error: ", error);
    return error;
  });
Se a mensagem for enviada com sucesso, o Twilio retorna um sendWhatsapp.sid, que é salvo no Firestore.
Se houver um erro, ele captura e imprime no console.
Problemas e Soluções
Você relatou três tipos de erro:

1. Erro 63024 - "Invalid message recipient"
🔍 Causa: O número de telefone do destinatário é inválido.
✅ Solução:

Confirme se o número está no formato internacional correto (whatsapp:+55XXXXXXXXXXX).
Verifique se o número tem conta no WhatsApp (adicione no seu WhatsApp e veja se aparece).
Se o número for fixo, ele não pode receber mensagens do WhatsApp Business.
2. Erro 63049 - "Meta chose not to deliver this WhatsApp marketing message"
🔍 Causa: O Meta bloqueou a entrega da mensagem porque pode ser considerada marketing.
✅ Solução:

Verifique se a mensagem está de acordo com as diretrizes do WhatsApp.
Use templates aprovados pelo WhatsApp em vez de mensagens personalizadas.
Peça para o usuário enviar uma mensagem para você primeiro, pois isso pode permitir o envio.
3. Erro 63032 - "User's number is part of an experiment"
🔍 Causa: O número do destinatário está participando de um experimento do WhatsApp e, por isso, não pode receber mensagens.
✅ Solução:

Tente outro número e veja se o problema persiste.
Espere algumas horas e tente novamente.
Entre em contato com o suporte do Twilio, pois esse erro pode ser temporário.
Melhorias no Código
Validar o número antes de enviar a mensagem

js
Copiar
Editar
if (!ownersPhones.startsWith("+55") || ownersPhones.length < 13) {
    console.error("Número inválido:", ownersPhones);
    return Promise.resolve();
}
🔹 Isso evita envios desnecessários para números incorretos.

Lidar melhor com erros do Twilio

js
Copiar
Editar
.catch((error) => {
    console.error("Erro ao enviar mensagem:", error);
    db.collection("errors").add({
        error: error.message,
        phone: ownersPhones,
        time: admin.firestore.FieldValue.serverTimestamp(),
    });
    return error;
});
🔹 Isso registra os erros no Firestore para facilitar o monitoramento.

Resumo
📌 Esse código escuta Firestore, pega os dados e envia mensagens via Twilio WhatsApp API.
⚠️ Os erros estão relacionados a números inválidos, restrições da Meta e experimentos do WhatsApp.
🚀 Soluções incluem validação do número, uso de templates e registro de erros no Firestore.

Se os erros persistirem, pode ser necessário entrar em contato com o suporte do Twilio para esclarecimentos adicionais. 🚀
