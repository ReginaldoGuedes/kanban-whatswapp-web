# kanban-whatswapp-web
CRM Kanbam  para WhatsApp Web Crhome tampermonkey como servidor 
abra o crhome http://tampermonkey.net/ 
instale a extensão tampermonkey > Ative a extensão > Clique em Adicionar novo script > copie o código abaixo.
Só irá funcionar com WhatsApp Web aberto aparecerá um borão azul CRM.

// ==UserScript==
// @name         CRM Kanban para WhatsApp Web
// @namespace    http://tampermonkey.net/
// @version      1.11
// @description  Organize contatos no WhatsApp Web em um CRM estilo Kanban!
// @author       Reginaldo Guedes
// @match        https://web.whatsapp.com/
// @grant        GM_setValue
// @grant        GM_getValue
// ==/UserScript==

(function() {
    'use strict';

    function createToggleButton() {
        let toggleButton = document.createElement('button');
        toggleButton.innerText = 'CRM';
        toggleButton.id = 'crmToggleButton';
        toggleButton.style.position = 'fixed';
        toggleButton.style.top = '10px';
        toggleButton.style.right = '10px';
        toggleButton.style.padding = '10px';
        toggleButton.style.background = '#007bff';
        toggleButton.style.color = '#fff';
        toggleButton.style.border = 'none';
        toggleButton.style.borderRadius = '5px';
        toggleButton.style.cursor = 'pointer';
        toggleButton.style.zIndex = '1000';
        toggleButton.onclick = toggleCRMBoard;
        document.body.appendChild(toggleButton);
    }

    function createCRMBoard() {
        if (document.getElementById('crmBoard')) return;

        let board = document.createElement('div');
        board.id = 'crmBoard';
        board.style.position = 'fixed';
        board.style.top = '60px';
        board.style.left = '10px';
        board.style.width = '95%';
        board.style.height = '75vh';
        board.style.background = '#f5f5f5';
        board.style.display = 'none';
        board.style.flexDirection = 'column';
        board.style.overflowX = 'auto';
        board.style.padding = '10px';
        board.style.gap = '10px';
        board.style.zIndex = '999';
        board.style.border = '1px solid #ccc';
        board.style.borderRadius = '8px';
        board.style.boxShadow = '2px 2px 10px rgba(0,0,0,0.2)';

        let searchContainer = document.createElement('div');
        searchContainer.style.padding = '10px';
        searchContainer.style.display = 'flex';
        searchContainer.style.justifyContent = 'center';

        let searchInput = document.createElement('input');
        searchInput.type = 'text';
        searchInput.placeholder = 'Adicionar contato manualmente';
        searchInput.style.padding = '8px';
        searchInput.style.border = '1px solid #ccc';
        searchInput.style.borderRadius = '5px';
        searchInput.style.marginRight = '5px';

        let addButton = document.createElement('button');
        addButton.innerText = 'Adicionar';
        addButton.style.padding = '8px';
        addButton.style.border = 'none';
        addButton.style.borderRadius = '5px';
        addButton.style.background = '#28a745';
        addButton.style.color = '#fff';
        addButton.style.cursor = 'pointer';
        addButton.onclick = () => {
            if (searchInput.value.trim() !== '') {
                let contact = createDraggableContact(searchInput.value.trim());
                containers['Negociação'].appendChild(contact);
                saveContacts();
                searchInput.value = '';
            }
        };

        searchContainer.appendChild(searchInput);
        searchContainer.appendChild(addButton);
        board.appendChild(searchContainer);

        let stages = [
            { name: 'Negociação', color: '#ff9800' },
            { name: 'Criando', color: '#03a9f4' },
            { name: 'Aprovado', color: '#4caf50' },
            { name: 'Pagamento', color: '#9c27b0' },
            { name: 'Em Produção', color: '#795548' },
            { name: 'Em Transporte', color: '#8bc34a' },
            { name: 'Entregue', color: '#f44336' }
        ];

        let containers = {};
        let columnsContainer = document.createElement('div');
        columnsContainer.style.display = 'flex';
        columnsContainer.style.gap = '10px';
        columnsContainer.style.overflowX = 'auto';
        board.appendChild(columnsContainer);

        stages.forEach(stage => {
            let column = document.createElement('div');
            column.classList.add('crmColumn');
            column.style.flex = '1';
            column.style.minWidth = '150px'; // Ajuste a largura mínima das colunas
            column.style.background = stage.color;
            column.style.border = '1px solid #ccc';
            column.style.borderRadius = '8px';
            column.style.padding = '10px';
            column.style.display = 'flex';
            column.style.flexDirection = 'column';
            column.style.gap = '5px';
            column.setAttribute('data-stage', stage.name);
            column.ondragover = (e) => e.preventDefault();
            column.ondrop = (e) => {
                let contactId = e.dataTransfer.getData('text');
                let contact = document.getElementById(contactId);
                if (contact) {
                    column.appendChild(contact);
                    saveContacts();
                }
            };

            let title = document.createElement('div');
            title.innerText = stage.name;
            title.style.fontWeight = 'bold';
            title.style.textAlign = 'center';
            title.style.paddingBottom = '5px';
            title.style.borderBottom = '2px solid #fff';
            title.style.color = '#000';
            column.appendChild(title);

            if (stage.name === 'Entregue') {
                let clearButton = document.createElement('button');
                clearButton.innerText = 'Deletar';
                clearButton.style.marginTop = '10px';
                clearButton.style.padding = '8px';
                clearButton.style.border = 'none';
                clearButton.style.borderRadius = '5px';
                clearButton.style.background = '#dc3545';
                clearButton.style.color = '#fff';
                clearButton.style.cursor = 'pointer';
                clearButton.onclick = () => {
                    column.querySelectorAll('.contact').forEach(contact => contact.remove());
                    saveContacts();
                };
                column.appendChild(clearButton);
            }

            columnsContainer.appendChild(column);
            containers[stage.name] = column;
        });

        document.body.appendChild(board);
        loadContacts();
    }

    function loadActiveChats() {
        let chatList = document.querySelectorAll("span[dir='auto']");
        chatList.forEach(chat => {
            let contactName = chat.innerText;
            let existingContact = document.getElementById(`contact-${contactName.replace(/\s/g, '')}`);
            if (!existingContact) {
                let contact = createDraggableContact(contactName);
                document.querySelector("[data-stage='Negociação']").appendChild(contact);
                saveContacts();
            }
        });
    }

    function toggleCRMBoard() {
        let board = document.getElementById('crmBoard');
        board.style.display = (board.style.display === 'none') ? 'flex' : 'none';
    }

    function createDraggableContact(name, observation = '') {
        let contact = document.createElement('div');
        contact.innerText = name;
        contact.classList.add('contact');
        contact.style.border = '1px solid #ccc';
        contact.style.padding = '8px';
        contact.style.borderRadius = '5px';
        contact.style.background = '#fff';
        contact.style.cursor = 'grab';
        contact.style.color = '#000';
        contact.style.overflow = 'hidden'; // Garantir que o conteúdo não ultrapasse os limites do contato
        contact.draggable = true;
        contact.style.boxShadow = '2px 2px 5px rgba(0,0,0,0.1)';
        contact.id = `contact-${name.replace(/\s/g, '')}`;
        contact.ondragstart = (e) => e.dataTransfer.setData('text', contact.id);

        // Evento de duplo clique para adicionar observação
        contact.ondblclick = () => {
            let observation = prompt('Digite sua observação:', contact.dataset.observation || '');
            if (observation !== null) {
                contact.dataset.observation = observation;
                let observationDisplay = contact.querySelector('.observation');
                if (!observationDisplay) {
                    observationDisplay = document.createElement('pre');
                    observationDisplay.classList.add('observation');
                    observationDisplay.style.marginTop = '5px';
                    observationDisplay.style.fontStyle = 'italic';
                    observationDisplay.style.color = '#555';
                    observationDisplay.style.whiteSpace = 'pre-wrap'; // Preservar quebras de linha
                    observationDisplay.style.wordWrap = 'break-word'; // Quebrar palavras longas
                    contact.appendChild(observationDisplay);
                }
                observationDisplay.innerText = observation;
                saveContacts();
            }
        };

        // Evento de triplo clique para editar observação
        let clickCount = 0;
        contact.addEventListener('click', () => {
            clickCount++;
            if (clickCount === 3) {
                let observation = prompt('Edite sua observação:', contact.dataset.observation || '');
                if (observation !== null) {
                    contact.dataset.observation = observation;
                    let observationDisplay = contact.querySelector('.observation');
                    if (!observationDisplay) {
                        observationDisplay = document.createElement('pre');
                        observationDisplay.classList.add('observation');
                        observationDisplay.style.marginTop = '5px';
                        observationDisplay.style.fontStyle = 'italic';
                        observationDisplay.style.color = '#555';
                        observationDisplay.style.whiteSpace = 'pre-wrap'; // Preservar quebras de linha
                        observationDisplay.style.wordWrap = 'break-word'; // Quebrar palavras longas
                        contact.appendChild(observationDisplay);
                    }
                    observationDisplay.innerText = observation;
                    saveContacts();
                }
                clickCount = 0;
            }
            setTimeout(() => {
                clickCount = 0;
            }, 500);
        });

        if (observation) {
            let observationDisplay = document.createElement('pre');
            observationDisplay.classList.add('observation');
            observationDisplay.style.marginTop = '5px';
            observationDisplay.style.fontStyle = 'italic';
            observationDisplay.style.color = '#555';
            observationDisplay.style.whiteSpace = 'pre-wrap'; // Preservar quebras de linha
            observationDisplay.style.wordWrap = 'break-word'; // Quebrar palavras longas
            observationDisplay.innerText = observation;
            contact.appendChild(observationDisplay);
        }

        return contact;
    }

    function saveContacts() {
        let contactsData = {};
        document.querySelectorAll('.crmColumn').forEach(column => {
            let stage = column.getAttribute('data-stage');
            contactsData[stage] = [];
            column.querySelectorAll('.contact').forEach(contact => {
                contactsData[stage].push({
                    name: contact.childNodes[0].nodeValue.trim(),
                    observation: contact.dataset.observation || ''
                });
            });
        });
        GM_setValue('crmContacts', JSON.stringify(contactsData));
    }

    function loadContacts() {
        let contactsData = JSON.parse(GM_getValue('crmContacts', '{}'));
        Object.keys(contactsData).forEach(stage => {
            let column = document.querySelector(`[data-stage='${stage}']`);
            contactsData[stage].forEach(contactData => {
                let contact = createDraggableContact(contactData.name, contactData.observation);
                column.appendChild(contact);
            });
        });
    }

    setTimeout(() => {
        createToggleButton();
        createCRMBoard();
    }, 5000);
})();
