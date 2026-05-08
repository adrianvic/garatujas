# core.js

```typescript
// Define o caminho absoluto do arquivo JSON de dados temporários na mesma pasta do script
const jsonFilePath = __dirname + '/data.temp.json';

// Declara uma variável array de strings que armazenará os itens da lista
// Inicializa carregando os dados do arquivo
const list: string[] = await loadFromFile();


// Define uma função que carrega dados do arquivo JSON
async function loadFromFile() {
  try {
    // Cria um objeto do arquivo JSON usando a API do Bun
    const file = Bun.file(jsonFilePath);
    
    // Lê o conteúdo do arquivo
    const content = await file.text();
    
    // Converte o texto JSON em um array de strings e retorna
    return JSON.parse(content) as string[];
  } catch (error: any) {
    // Verifica se o erro é de arquivo não encontrado
    if (error.code === 'ENOENT')
      // Se o arquivo não existe, retorna um array vazio
      return [];
    
    // Se for outro tipo de erro, lança exceção
    throw error;
  }
}


// Define uma função assíncrona que salva a lista no arquivo JSON
async function saveToFile() {
  try {
    // Escreve o conteúdo da variável list convertido em JSON no arquivo
    // Bun.write grava o arquivo
    await Bun.write(jsonFilePath, JSON.stringify(list));
  } catch (error: any) {
    // Se der erro lança uma exceção
    throw new Error("Erro ao salvar os dados no arquivo: " + error.message);
  }
}


// Define uma função que adiciona um novo item à lista
async function addItem(item: string) {
  // Adiciona o item ao final do array
  list.push(item);
  
  // Salva a lista atualizada no arquivo
  await saveToFile();
}


// Define uma função assíncrona que retorna todos os itens da lista
async function getItems() {
  // Retorna array completo de itens
  return list;
}


// Define função que atualiza um item em um índice específico
async function updateItem(index: number, newItem: string) {
  // Valida se o índice está dentro dos limites do array
  if (index < 0 || index >= list.length)
    // Se inválido lança uma exceção
    throw new Error("Index fora dos limites");
  
  // Substitui o item no índice especificado
  list[index] = newItem;
  
  // Salva a lista atualizada
  await saveToFile();
}


// Define uma função que remove um item em um índice específico
async function removeItem(index: number) {
  // Valida se o índice está dentro dos limites válidos do array
  if (index < 0 || index >= list.length)
    // Se inválido lança uma exceção
    throw new Error("Index fora dos limites");
  
  // Remove 1 elemento a partir do índice especificado
  list.splice(index, 1);
  
  // Salva a lista atualizada
  await saveToFile();
}


// Exporta um objeto contendo todas as funções
export default { addItem, getItems, updateItem, removeItem };
```

# api.turma2.ts

```typescript
// Importa o módulo de gerenciamento de tarefas do arquivo core.ts
import todo from "./core.ts";

// Inicia um servidor HTTP usando Bun na porta 3000
const server = Bun.serve({
  port: 3000,

  // Rotas e handlers HTTP do servidor
  routes: {
    // Rota raiz  retorna um arquivo HTML estático
    "/": new Response(Bun.file("./public/index.html")),

    // Rota da API para gerenciar tarefas
    "/api/todo": {
      // retorna a lista de todas as tarefas
      GET: async () => {
        // Chama a função para obter todos os itens
        const items = await todo.getItems()
        
        // Retorna os itens como resposta JSON
        return Response.json(items)
      },

      // adiciona uma nova tarefa à lista
      POST: async (req) => {
        // Extrai o corpo da requisição como JSON
        const data = await req.json() as any;
        
        // Obtém a propriedade item dos dados recebidos, ou null se não existir
        const item = data.item || null;
        
        // Valida se o campo 'item' foi fornecido
        if (!item)
          // Se não foi fornecido, retorna erro 400
          return Response.json('Por favor, forneça um item para adicionar.', { status: 400 });
        
        // Adiciona o item à lista
        await todo.addItem(item);
        
        // Retorna os dados recebidos como confirmação
        return Response.json(data);
      },
    },

    // Rota para atualizar ou deletar uma tarefa pelo índice
    "/api/todo/:index": {
      // atualiza uma tarefa existente
      PUT: async (req) => {
        // Extrai a propriedade index da URL e converte para número
        const index = parseInt(req.params.index);
        
        // Valida se o índice é um número válido
        if (isNaN(index))
          // Se não for válido retorna erro 400
          return Response.json('Índice inválido. um número inteiro é esperado.', { status: 400 });
        
        // Extrai o corpo da requisição como JSON
        const data = await req.json() as any;
        
        // Obtém a propriedade newItem dos dados, ou null se não existir
        const newItem = data.newItem || null;
        
        // Valida se o campo newItem existe
        if (!newItem)
          // Se não foi fornecido retorna erro 400
          return Response.json('Por favor, forneça um novo item para atualizar.', { status: 400 });
        
        try {
          // Tenta atualizar o item no índice especificado
          await todo.updateItem(index, newItem);
          
          // Retorna mensagem de sucesso
          return Response.json(`Item no índice ${index} atualizado para "${newItem}".`);
        } catch (error: any) {
          // Se ocorrer erro retorna erro 400 com mensagem
          return Response.json(error.message, { status: 400 });
        }
      },

      // remove uma tarefa específica
      DELETE: async (req) => {
        // Extrai o parâmetro index da URL e converte para número inteiro
        const index = parseInt(req.params.index);
        
        // Valida se o índice é um número válido
        if (isNaN(index))
          // Se não for válido, retorna erro 400 com mensagem em português
          return Response.json('Índice inválido.', { status: 400 });
        
        try {
          // Tenta remover o item no índice especificado
          await todo.removeItem(index);
          
          // Retorna mensagem de sucesso
          return Response.json(`Item no índice ${index} removido com sucesso.`);
        } catch (error: any) {
          // Se ocorrer erro retorna erro 400 com mensagem
          return Response.json(error.message, { status: 400 });
        }
      },
    },
  },

  // Catch all padrão para rotas não definidas
  async fetch(req) {
    // Retorna erro 404 (Não Encontrado) para qualquer rota não mapeada
    return new Response(`Not Found`, { status: 404 });
  },
});

// Imprime no console a mensagem de que o servidor está rodando e em qual URL
console.log(`Server running at http://localhost:${server.port}`);
```
