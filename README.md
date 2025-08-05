import sys
import sqlite3
from PyQt6.QtWidgets import (
    QApplication, QWidget, QVBoxLayout, QHBoxLayout,
    QListWidget, QLineEdit, QTextEdit, QPushButton, QMessageBox
)

class NoteManager(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Менеджер Заметок")
        self.resize(800, 400)

        self.conn = sqlite3.connect("notes.db")
        self.cursor = self.conn.cursor()
        self.create_table()

        self.setup_ui()
        self.load_notes()

    def create_table(self):
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS notes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                content TEXT,
                created_at TEXT DEFAULT CURRENT_TIMESTAMP
            )
        """)
        self.conn.commit()

    def setup_ui(self):
        main_layout = QHBoxLayout()

        # Список замето
        self.note_list = QListWidget()
        self.note_list.currentItemChanged.connect(self.load_selected_note)
        main_layout.addWidget(self.note_list, 2)

        # Правя часть: поля редактирования
        edit_layout = QVBoxLayout()

        self.title_input = QLineEdit()
        self.title_input.setPlaceholderText("Заголовок")

        self.content_input = QTextEdit()
        self.content_input.setPlaceholderText("Содержание заметки")

        # Кнопки
        self.add_btn = QPushButton("Добавить")
        self.save_btn = QPushButton("Сохранить")
        self.delete_btn = QPushButton("Удалить")

        self.add_btn.clicked.connect(self.add_note)
        self.save_btn.clicked.connect(self.save_note)
        self.delete_btn.clicked.connect(self.delete_note)

        # Добавлене в layout
        edit_layout.addWidget(self.title_input)
        edit_layout.addWidget(self.content_input)
        edit_layout.addWidget(self.add_btn)
        edit_layout.addWidget(self.save_btn)
        edit_layout.addWidget(self.delete_btn)

        main_layout.addLayout(edit_layout, 3)
        self.setLayout(main_layout)

    def load_notes(self):
        self.note_list.clear()
        self.cursor.execute("SELECT id, title FROM notes ORDER BY created_at DESC")
        for note_id, title in self.cursor.fetchall():
            self.note_list.addItem(f"{note_id}: {title}")

    def get_selected_note_id(self):
        item = self.note_list.currentItem()
        if item:
            return int(item.text().split(":")[0])
        return None

    def load_selected_note(self):
        note_id = self.get_selected_note_id()
        if note_id:
            self.cursor.execute("SELECT title, content FROM notes WHERE id = ?", (note_id,))
            note = self.cursor.fetchone()
            if note:
                self.title_input.setText(note[0])
                self.content_input.setText(note[1])
        else:
            self.title_input.clear()
            self.content_input.clear()

    def add_note(self):
        title = self.title_input.text().strip()
        content = self.content_input.toPlainText().strip()

        if not title:
            QMessageBox.warning(self, "Ошибка", "Заголовок не может быть пустым.")
            return

        self.cursor.execute("INSERT INTO notes (title, content) VALUES (?, ?)", (title, content))
        self.conn.commit()
        self.load_notes()
        self.title_input.clear()
        self.content_input.clear()

    def save_note(self):
        note_id = self.get_selected_note_id()
        if not note_id:
            QMessageBox.warning(self, "Ошибка", "Сначала выберите заметку для редактирования.")
            return

        title = self.title_input.text().strip()
        content = self.content_input.toPlainText().strip()

        if not title:
            QMessageBox.warning(self, "Ошибка", "Заголовок не может быть пустым.")
            return

        self.cursor.execute("UPDATE notes SET title = ?, content = ? WHERE id = ?", (title, content, note_id))
        self.conn.commit()
        self.load_notes()

    def delete_note(self):
        note_id = self.get_selected_note_id()
        if not note_id:
            QMessageBox.warning(self, "Ошибка", "Выберите заметку для удаления.")
            return

        confirm = QMessageBox.question(
            self, "Удалить заметку",
            "Вы уверены, что хотите удалить эту заметку?",
            QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No
        )

        if confirm == QMessageBox.StandardButton.Yes:
            self.cursor.execute("DELETE FROM notes WHERE id = ?", (note_id,))
            self.conn.commit()
            self.load_notes()
            self.title_input.clear()
            self.content_input.clear()

    def closeEvent(self, event):
        self.conn.close()
        event.accept()


if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = NoteManager()
    window.show()
    sys.exit(app.exec())
