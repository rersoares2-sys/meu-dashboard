<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WIFI LOJAS FMW</title>
    <!-- Inclui a biblioteca Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #dbeafe;
        }
        .table-row-hover:hover {
            background-color: #f0f0f0;
            cursor: pointer;
        }
        .modal {
            display: none;
            position: fixed;
            z-index: 1000;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            overflow: auto;
            background-color: rgba(0,0,0,0.4);
            justify-content: center;
            align-items: center;
        }
        .btn-fmw {
            background-color: #212121;
            color: white;
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
            transition: background-color 0.3s ease, transform 0.2s ease;
        }
        .btn-fmw:hover {
            background-color: #424242;
            transform: translateY(-2px);
        }
    </style>
</head>
<body class="p-4 sm:p-8">

    <div class="max-w-7xl mx-auto bg-white p-6 sm:p-10 rounded-xl shadow-2xl">
        <h1 class="text-3xl sm:text-4xl font-extrabold text-gray-900 text-center mb-2">WIFI LOJAS FMW</h1>
        <p class="text-center text-gray-500 mb-8">Essa tela é para consultar as senhas de wifi de cada loja da camisaria FMW</p>
        
        <!-- Controles de Filtro e Botão de Atualização -->
        <div class="flex flex-col md:flex-row gap-4 mb-8">
            <input type="text" id="filter-id" placeholder="Filtrar por nome da loja..." class="p-3 w-full border border-gray-300 rounded-lg focus:ring-2 focus:ring-gray-400 focus:border-transparent">
            <input type="text" id="filter-name" placeholder="Filtrar por nome da rede..." class="p-3 w-full border border-gray-300 rounded-lg focus:ring-2 focus:ring-gray-400 focus:border-transparent">
            <input type="text" id="filter-country" placeholder="Filtrar por senha..." class="p-3 w-full border border-gray-300 rounded-lg focus:ring-2 focus:ring-gray-400 focus:border-transparent">
            <button id="refresh-button" class="p-3 w-full md:w-auto rounded-lg font-semibold btn-fmw">
                Atualizar Dados
            </button>
        </div>

        <!-- Área de notificação de alteração -->
        <div id="change-notification" class="hidden px-4 py-3 rounded relative mb-4" role="alert">
            <strong class="font-bold" id="notification-strong"></strong>
            <span class="block sm:inline" id="notification-message"></span>
        </div>

        <!-- Área de exibição da tabela -->
        <div class="overflow-x-auto shadow-md rounded-lg">
            <table class="w-full text-sm text-left text-gray-500">
                <thead class="text-xs text-gray-700 uppercase bg-gray-100">
                    <tr>
                        <th scope="col" class="px-6 py-3">Nome da Loja</th>
                        <th scope="col" class="px-6 py-3">Nome da Rede</th>
                        <th scope="col" class="px-6 py-3">Senha</th>
                    </tr>
                </thead>
                <tbody id="data-table-body">
                    <!-- A linha de carregamento será adicionada dinamicamente -->
                </tbody>
            </table>
        </div>
        
        <div id="no-results" class="hidden text-center text-lg font-medium text-gray-600 mt-8">
            Nenhum resultado encontrado para os filtros aplicados.
        </div>

        <!-- Modal para detalhes do aluno -->
        <div id="student-modal" class="modal">
            <div class="bg-white rounded-lg shadow-xl p-6 w-11/12 md:w-2/3 lg:w-1/2">
                <div class="flex justify-between items-center border-b pb-3 mb-4">
                    <h3 class="text-2xl font-bold text-gray-900" id="modal-title">Detalhes da Loja</h3>
                    <button class="text-gray-500 hover:text-gray-700" id="close-modal-btn">
                        <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path></svg>
                    </button>
                </div>
                <div id="modal-content" class="text-gray-700 space-y-2">
                    <!-- Detalhes da loja serão injetados aqui -->
                </div>
            </div>
        </div>
    </div>

    <script>
        const SPREADSHEET_ID = '1cOkiecNuAKR_gmcULGd29_NK8S_vtuBj8UOgZHWS7YI';
        const GID = '578667057';
        const GOOGLE_SHEET_URL = `https://docs.google.com/spreadsheets/d/${SPREADSHEET_ID}/gviz/tq?tqx=out:json&gid=${GID}`;

        const COLUMN_MAP = {
            'ID': 0,
            'NOME': 1,
            'PAÍS': 2,
            'E-MAIL': 3,
            'GÊNERO': 4,
            'IDADE': 5,
            'UNIVERSIDADE': 6,
            'GRAU': 7,
            'DATA DE INSCRIÇÃO': 8,
            'DATA DA ÚLTIMA ATUALIZAÇÃO': 9
        };

        let allData = [];
        let previousData = [];

        function parseSheetDate(dateString) {
            if (!dateString) return 'N/A';
            try {
                const excelEpoch = new Date('1899-12-30T00:00:00Z');
                const msPerDay = 24 * 60 * 60 * 1000;
                const date = new Date(excelEpoch.getTime() + (parseFloat(dateString) - 1) * msPerDay);
                return date.toLocaleDateString('pt-BR');
            } catch (e) {
                console.error("Erro ao converter data:", e);
                return dateString;
            }
        }

        async function fetchData() {
            const tableBody = document.getElementById('data-table-body');
            
            // Oculta a notificação antes de cada busca
            hideNotification();
            
            // Adiciona a linha de carregamento dinamicamente
            tableBody.innerHTML = `
                <tr id="loading" class="bg-white border-b">
                    <td colspan="3" class="px-6 py-4 text-center text-gray-600 animate-pulse">Carregando dados...</td>
                </tr>
            `;
            
            // Salva os dados atuais para comparação
            previousData = [...allData];

            try {
                const response = await fetch(GOOGLE_SHEET_URL);
                const text = await response.text();
                
                const jsonStart = text.indexOf('(') + 1;
                const jsonEnd = text.lastIndexOf(')');
                const jsonString = text.substring(jsonStart, jsonEnd);
                const parsedJson = JSON.parse(jsonString);
                
                const rows = parsedJson.table.rows;
                
                const newData = rows.map(row => {
                    const c = row.c;
                    return {
                        id: c[COLUMN_MAP['ID']]?.v,
                        name: c[COLUMN_MAP['NOME']]?.v,
                        country: c[COLUMN_MAP['PAÍS']]?.v,
                        email: c[COLUMN_MAP['E-MAIL']]?.v,
                        gender: c[COLUMN_MAP['GÊNERO']]?.v,
                        age: c[COLUMN_MAP['IDADE']]?.v,
                        university: c[COLUMN_MAP['UNIVERSIDADE']]?.v,
                        degree: c[COLUMN_MAP['GRAU']]?.v,
                        enrollment_date: parseSheetDate(c[COLUMN_MAP['DATA DE INSCRIÇÃO']]?.v),
                        last_update_date: parseSheetDate(c[COLUMN_MAP['DATA DA ÚLTIMA ATUALIZAÇÃO']]?.v)
                    };
                });
                
                // Remove a primeira linha da planilha.
                allData = newData.slice(1);
                
                // Compara os dados para encontrar alterações
                checkForChanges();

                renderData(allData);

            } catch (error) {
                console.error('Erro ao buscar ou processar os dados da planilha:', error);
                tableBody.innerHTML = `<td colspan="3" class="px-6 py-4 text-center text-red-500">Erro ao carregar os dados.</td>`;
            }
        }

        function checkForChanges() {
            const notificationDiv = document.getElementById('change-notification');
            const notificationStrong = document.getElementById('notification-strong');
            const notificationMessage = document.getElementById('notification-message');
            
            // Converte para strings para uma comparação mais fácil
            const previousDataString = JSON.stringify(previousData);
            const newDataString = JSON.stringify(allData);

            if (previousData.length === 0) {
                // Primeira carga de dados, não mostra notificação de alteração
                return;
            }

            if (previousDataString !== newDataString) {
                let changeFound = false;
                for (let i = 0; i < allData.length; i++) {
                    // Verifica se a linha existe nos dados antigos para evitar erros
                    if (previousData[i]) {
                        // Compara cada campo para encontrar a alteração
                        if (allData[i].id !== previousData[i].id || allData[i].name !== previousData[i].name || allData[i].country !== previousData[i].country) {
                            notificationStrong.textContent = 'Atenção!';
                            notificationMessage.textContent = `A loja "${allData[i].id || 'N/A'}" teve informações alteradas.`;
                            showNotification('bg-yellow-100 border-yellow-400 text-yellow-700');
                            changeFound = true;
                            break; // Sai do loop após encontrar a primeira alteração
                        }
                    }
                }
                
                // Caso não encontre uma alteração específica, mas o número de linhas seja diferente
                if (!changeFound && allData.length !== previousData.length) {
                    notificationStrong.textContent = 'Atenção!';
                    notificationMessage.textContent = `O número de lojas na planilha foi alterado.`;
                    showNotification('bg-yellow-100 border-yellow-400 text-yellow-700');
                }
            } else {
                // Nenhuma alteração encontrada
                notificationStrong.textContent = 'Atualizado!';
                notificationMessage.textContent = `Nenhuma nova atualização encontrada.`;
                showNotification('bg-green-100 border-green-400 text-green-700');
            }
        }

        function showNotification(bgColor) {
            const notificationDiv = document.getElementById('change-notification');
            notificationDiv.className = `px-4 py-3 rounded relative mb-4 border ${bgColor}`;
            notificationDiv.classList.remove('hidden');
            
            // Oculta a notificação após 5 segundos
            setTimeout(hideNotification, 5000);
        }
        
        function hideNotification() {
            document.getElementById('change-notification').classList.add('hidden');
        }

        function renderData(data) {
            const tableBody = document.getElementById('data-table-body');
            tableBody.innerHTML = '';
            
            if (data.length === 0) {
                document.getElementById('no-results').classList.remove('hidden');
                return;
            }
            document.getElementById('no-results').classList.add('hidden');

            data.forEach((item, index) => {
                const row = document.createElement('tr');
                row.classList.add('bg-white', 'border-b', 'table-row-hover');
                row.innerHTML = `
                    <td class="px-6 py-4 font-medium text-gray-900 whitespace-nowrap">${item.id || 'N/A'}</td>
                    <td class="px-6 py-4 font-medium text-gray-900 whitespace-nowrap">${item.name || 'N/A'}</td>
                    <td class="px-6 py-4">${item.country || 'N/A'}</td>
                `;
                row.addEventListener('click', () => showDetailsModal(item));
                tableBody.appendChild(row);
            });
        }

        function filterData() {
            const idFilter = document.getElementById('filter-id').value.toLowerCase();
            const nameFilter = document.getElementById('filter-name').value.toLowerCase();
            const countryFilter = document.getElementById('filter-country').value.toLowerCase();

            const filteredData = allData.filter(item => {
                const idMatch = !idFilter || (item.id && item.id.toString().toLowerCase().includes(idFilter));
                const nameMatch = !nameFilter || (item.name && item.name.toLowerCase().includes(nameFilter));
                const countryMatch = !countryFilter || (item.country && item.country.toLowerCase().includes(countryFilter));
                
                return idMatch && nameMatch && countryMatch;
            });

            renderData(filteredData);
        }
        
        function showDetailsModal(item) {
            const modal = document.getElementById('student-modal');
            const modalContent = document.getElementById('modal-content');
            document.getElementById('modal-title').textContent = `Detalhes da Loja: ${item.name || 'N/A'}`;
            
            modalContent.innerHTML = `
                <p><strong>Nome da Loja:</strong> ${item.id || 'N/A'}</p>
                <p><strong>Nome da Rede:</strong> ${item.name || 'N/A'}</p>
                <p><strong>Senha:</strong> ${item.country || 'N/A'}</p>
            `;
            modal.style.display = 'flex';
        }

        document.getElementById('close-modal-btn').addEventListener('click', () => {
            document.getElementById('student-modal').style.display = 'none';
        });

        document.getElementById('student-modal').addEventListener('click', (event) => {
            if (event.target === document.getElementById('student-modal')) {
                document.getElementById('student-modal').style.display = 'none';
            }
        });

        document.getElementById('filter-id').addEventListener('input', filterData);
        document.getElementById('filter-name').addEventListener('input', filterData);
        document.getElementById('filter-country').addEventListener('input', filterData);
        document.getElementById('refresh-button').addEventListener('click', fetchData);
        
        // Chama a função para buscar os dados quando a página carrega
        fetchData();
        
        // Configura um intervalo para buscar os dados automaticamente a cada 30 minutos (1800000ms)
        setInterval(fetchData, 1800000);
    </script>

</body>
</html>
