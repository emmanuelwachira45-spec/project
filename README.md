<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>To-Do List (Selectable)</title>

<style>
body {
    font-family: Arial;
    display: flex;
    justify-content: center;
    margin-top: 50px;
}

.container {
    width: 400px;
}

input[type="text"] {
    padding: 8px;
    width: 65%;
}

button {
    padding: 8px;
    cursor: pointer;
}

table {
    width: 100%;
    margin-top: 15px;
    border-collapse: collapse;
}

th, td {
    padding: 10px;
    border-bottom: 1px solid #ddd;
}

.completed {
    text-decoration: line-through;
    color: gray;
}
</style>
</head>

<body>

<div class="container">
    <h2>To-Do List</h2>

    <form id="form">
        <input type="text" id="input" placeholder="Add task..." required>
        <button>Add</button>
    </form>

    <table>
        <thead>
            <tr>
                <th>Select</th>
                <th>Task</th>
            </tr>
        </thead>
        <tbody id="table-body"></tbody>
    </table>

    <button onclick="clearSelected()">Clear Selected</button>
</div>

<script>
let todos = [];

const form = document.getElementById("form");
const input = document.getElementById("input");
const tableBody = document.getElementById("table-body");

form.addEventListener("submit", function(e) {
    e.preventDefault();

    const text = input.value.trim();
    if (!text) return;

    todos.push({
        id: Date.now(),
        text,
        completed: false,
        selected: false
    });

    input.value = "";
    render();
});

function render() {
    tableBody.innerHTML = "";

    todos.forEach(todo => {
        const row = document.createElement("tr");

        row.innerHTML = `
            <td>
                <input type="checkbox" onchange="selectTask(${todo.id})" ${todo.selected ? "checked" : ""}>
            </td>
            <td class="${todo.completed ? 'completed' : ''}" onclick="toggle(${todo.id})">
                ${todo.text}
            </td>
        `;

        tableBody.appendChild(row);
    });
}

// Toggle complete
function toggle(id) {
    todos = todos.map(t =>
        t.id === id ? { ...t, completed: !t.completed } : t
    );
    render();
}

// Select task
function selectTask(id) {
    todos = todos.map(t =>
        t.id === id ? { ...t, selected: !t.selected } : t
    );
}

// Clear selected
function clearSelected() {
    todos = todos.filter(t => !t.selected);
    render();
}
</script>

</body>
</html>
