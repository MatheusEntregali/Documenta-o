<h1> notificationsViaWhatsappByParcel </h1>
Firebase Cloud Function: Notificações via WhatsApp
Este repositório contém uma função do Firebase Cloud Functions que escuta mudanças no Firestore e dispara mensagens de WhatsApp via Twilio quando um novo documento é adicionado na subcoleção /notificationsByWhatsapp/ dentro de /parcels/{parcel}.

Descrição do Código
O código é uma função do Firebase Cloud Functions que monitora alterações no Firestore e envia mensagens de WhatsApp utilizando a API do Twilio. A função é acionada quando um novo documento é criado na subcoleção /notificationsByWhatsapp/ dentro de /parcels/{parcel}.

Fluxo de Execução
Ativação da Função: A função é acionada quando um novo documento é criado em parcels/{parcel}/notificationsByWhatsapp/{notificationsByWhatsappId}.

Extração de Dados: O código extrai os dados do documento e formata o código de retirada da encomenda.

Verificação do Tipo de Notificação: Dependendo do tipo de notificação (variant), a função executa uma das seguintes ações:

Envia uma mensagem ao dono da encomenda (newParcelToOwner).

Envia uma mensagem ao administrador do locker (newParcelToAdmin).

Envia um lembrete ao dono (parcelReminderToOwner).

Envia um lembrete ao administrador (parcelReminderToAdmin).

Envio da Mensagem pelo Twilio: Se houver um número de telefone válido, a função chama a API do Twilio para enviar a mensagem via WhatsApp.

Salvamento do Recibo: Se o envio for bem-sucedido, um recibo da mensagem é salvo no Firestore na coleção notifications.

Tratamento de Erros: Se ocorrer um erro ao enviar a mensagem pelo Twilio, o erro é capturado e registrado no console.

Componentes Principais
1. Ativação da Função
javascript
Copy
exports.notificationsViaWhatsappByParcel = functions.firestore
  .document("/parcels/{parcel}/notificationsByWhatsapp/{notificationsByWhatsappId}")
  .onCreate((snap, context) => {
Esta função é um gatilho que escuta a criação de novos documentos na coleção notificationsByWhatsapp/ dentro de parcels/{parcel}.

2. Extração de Dados
javascript
Copy
const value = snap.data();
const parcel = context.params.parcel;
const db = admin.firestore();
const retrieveCode = value.retrieveCode;
const formatedRetrieveCode = retrieveCode.substr(0, 3).concat("-", retrieveCode.substr(3, 3));
Obtém os dados do documento (value) e formata o código de retirada para exibição no WhatsApp.

3. Verificação do Tipo de Notificação
Cada bloco verifica a propriedade variant e executa a ação correspondente.

javascript
Copy
if (value.variant === "newParcelToOwner") { ... }
if (value.variant === "newParcelToAdmin") { ... }
if (value.variant === "parcelReminderToOwner") { ... }
if (value.variant === "parcelReminderToAdmin") { ... }
Define variáveis como locker, unit, unitType, lockerName, ownersPhones, etc. Verifica se há um número de telefone válido para envio.

4. Envio da Mensagem pelo Twilio
Se o número for válido, ele envia uma mensagem via Twilio WhatsApp API.

javascript
Copy
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
Se a mensagem for enviada com sucesso, o Twilio retorna um sendWhatsapp.sid, que é salvo no Firestore. Se houver um erro, ele captura e imprime no console.

Resumo
📌 Esse código escuta Firestore, pega os dados e envia mensagens via Twilio WhatsApp API.

⚠️ Os erros estão relacionados a números inválidos, restrições da Meta e experimentos do WhatsApp.

Se os erros persistirem, pode ser necessário entrar em contato com o suporte do Twilio para esclarecimentos adicionais. 🚀

