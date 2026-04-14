import sys
import os
sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), '..')))

import pytest
from fastapi.testclient import TestClient
from main import app, save_todos, load_todos, TodoItem


client = TestClient(app)

# ===== Fixtures =====
@pytest.fixture(autouse=True)
def reset_todos():
    """각 테스트 전후에 todos 초기화"""
    save_todos([])
    yield
    save_todos([])


@pytest.fixture
def sample_todo():
    """테스트용 기본 Todo 객체"""
    return TodoItem(
        id=1,
        title="Test Todo",
        description="Test description",
        completed=False
    )


@pytest.fixture
def sample_todo_dict():
    """테스트용 기본 Todo 딕셔너리"""
    return {
        "id": 1,
        "title": "Test Todo",
        "description": "Test description",
        "completed": False
    }


def create_todo_in_db(todo: TodoItem) -> None:
    """데이터베이스에 todo 추가하는 헬퍼 함수"""
    todos = load_todos()
    todos.append(todo.dict())
    save_todos(todos)


# ===== GET Tests =====
class TestGetTodos:
    def test_get_empty_list(self):
        """빈 todo 리스트 조회"""
        response = client.get("/todos")
        assert response.status_code == 200
        assert response.json() == []

    def test_get_single_todo(self, sample_todo):
        """단일 todo 조회"""
        create_todo_in_db(sample_todo)
        
        response = client.get("/todos")
        assert response.status_code == 200
        data = response.json()
        assert len(data) == 1
        assert data[0]["title"] == "Test Todo"
        assert data[0]["completed"] is False

    def test_get_multiple_todos(self, sample_todo):
        """여러 todos 조회"""
        todos = [
            TodoItem(id=1, title="First", description="Desc1", completed=False),
            TodoItem(id=2, title="Second", description="Desc2", completed=True),
        ]
        for todo in todos:
            create_todo_in_db(todo)
        
        response = client.get("/todos")
        assert response.status_code == 200
        assert len(response.json()) == 2


# ===== POST Tests =====
class TestCreateTodo:
    def test_create_valid_todo(self, sample_todo_dict):
        """유효한 todo 생성"""
        response = client.post("/todos", json=sample_todo_dict)
        assert response.status_code == 200
        data = response.json()
        assert data["title"] == "Test Todo"
        assert data["description"] == "Test description"
        assert data["completed"] is False

    def test_create_todo_missing_required_field(self):
        """필수 필드 누락 - 422 Unprocessable Entity"""
        invalid_todo = {"id": 1, "title": "Test"}
        response = client.post("/todos", json=invalid_todo)
        assert response.status_code == 422

    def test_create_todo_empty_body(self):
        """빈 요청 바디"""
        response = client.post("/todos", json={})
        assert response.status_code == 422

    def test_create_todo_persisted(self, sample_todo_dict):
        """생성된 todo가 실제로 저장되었는지 확인"""
        client.post("/todos", json=sample_todo_dict)
        
        response = client.get("/todos")
        assert len(response.json()) == 1
        assert response.json()[0]["title"] == "Test Todo"


# ===== PUT Tests =====
class TestUpdateTodo:
    def test_update_existing_todo(self, sample_todo):
        """기존 todo 업데이트"""
        create_todo_in_db(sample_todo)
        
        updated_data = {
            "id": 1,
            "title": "Updated Title",
            "description": "Updated description",
            "completed": True
        }
        response = client.put("/todos/1", json=updated_data)
        
        assert response.status_code == 200
        data = response.json()
        assert data["title"] == "Updated Title"
        assert data["completed"] is True

    def test_update_nonexistent_todo(self):
        """존재하지 않는 todo 업데이트 시도"""
        updated_data = {
            "id": 999,
            "title": "Updated",
            "description": "Updated description",
            "completed": True
        }
        response = client.put("/todos/999", json=updated_data)
        assert response.status_code == 404

    def test_update_partial_fields(self, sample_todo):
        """특정 필드만 업데이트"""
        create_todo_in_db(sample_todo)
        
        updated_data = {
            "id": 1,
            "title": "Only Title Changed",
            "description": "Test description",
            "completed": False
        }
        response = client.put("/todos/1", json=updated_data)
        
        assert response.status_code == 200
        assert response.json()["title"] == "Only Title Changed"


# ===== DELETE Tests =====
class TestDeleteTodo:
    def test_delete_existing_todo(self, sample_todo):
        """기존 todo 삭제"""
        create_todo_in_db(sample_todo)
        
        response = client.delete("/todos/1")
        assert response.status_code == 200
        assert response.json()["message"] == "To-Do item deleted"

    def test_delete_todo_actually_removed(self, sample_todo):
        """삭제 후 실제로 데이터베이스에서 제거되었는지 확인"""
        create_todo_in_db(sample_todo)
        
        client.delete("/todos/1")
        response = client.get("/todos")
        
        assert len(response.json()) == 0

    def test_delete_nonexistent_todo(self):
        """존재하지 않는 todo 삭제 시도 - 404 반환"""
        response = client.delete("/todos/999")
        assert response.status_code == 404  # 버그 수정: 200이 아니라 404

    def test_delete_multiple_todos(self):
        """여러 todos 중 특정 todo만 삭제"""
        todos = [
            TodoItem(id=1, title="First", description="Desc1", completed=False),
            TodoItem(id=2, title="Second", description="Desc2", completed=False),
        ]
        for todo in todos:
            create_todo_in_db(todo)
        
        # ID 1 삭제
        response = client.delete("/todos/1")
        assert response.status_code == 200
        
        # ID 2만 남아있는지 확인
        todos_list = client.get("/todos").json()
        assert len(todos_list) == 1
        assert todos_list[0]["id"] == 2


# ===== Edge Cases =====
class TestEdgeCases:
    def test_todo_with_special_characters(self):
        """특수문자가 포함된 todo"""
        todo = {
            "id": 1,
            "title": "Test!@#$%^&*()",
            "description": "한글 테스트 😀",
            "completed": False
        }
        response = client.post("/todos", json=todo)
        assert response.status_code == 200
        assert response.json()["title"] == "Test!@#$%^&*()"

    def test_todo_with_long_title(self):
        """긴 제목의 todo"""
        long_title = "A" * 500
        todo = {
            "id": 1,
            "title": long_title,
            "description": "Test",
            "completed": False
        }
        response = client.post("/todos", json=todo)
        assert response.status_code == 200

    def test_concurrent_ids(self):
        """ID 충돌 처리"""
        todo1 = {
            "id": 1,
            "title": "First",
            "description": "Desc1",
            "completed": False
        }
        todo2 = {
            "id": 1,
            "title": "Second",
            "description": "Desc2",
            "completed": False
        }
        
        client.post("/todos", json=todo1)
        response = client.post("/todos", json=todo2)
        
        # 중복 ID에 대한 처리 (구현에 따라 다를 수 있음)
        assert response.status_code in [200, 400, 409]