import sys
import os
from PyQt6.QtWidgets import (
    QApplication, QDialog, QMessageBox, QTextEdit, QLineEdit, QPushButton, QWidget
)
from PyQt6.uic import loadUi
from google import genai
from google.genai.errors import APIError

class GeminiChatApp(QDialog):
    """
    kang_gemini.ui 파일을 로드하고 Gemini API와 통신하는 채팅 애플리케이션 클래스
    """
    # API 키 설정
    GEMINI_API_KEY = "GEMINI_API_KEY"
    
    def __init__(self):
        super().__init__()
        
        # 1. UI 파일 로드
        try:
            # loadUi를 사용하여 UI 파일의 내용을 self (QDialog)에 적용합니다.
            loadUi("kang_gemini.ui", self)
        except FileNotFoundError:
            QMessageBox.critical(self, "오류", 
                                 "kang_gemini.ui 파일을 찾을 수 없습니다. 파일 경로를 확인해주세요.")
            sys.exit(1)

        # 2. 위젯 연결 (children() 메서드를 사용하여 모든 위젯을 찾아 연결)
        # loadUi가 실패했을 경우를 대비하여 모든 자식 위젯을 순회하며 objectName으로 찾습니다.
        self.answer_output = None
        self.question_input = None
        self.send_button = None
        
        # 모든 자식 위젯(children())을 순회합니다.
        for child in self.findChildren(QWidget):
            obj_name = child.objectName()
            
            if obj_name == 'answer' and isinstance(child, QTextEdit):
                self.answer_output = child
            elif obj_name == 'question' and isinstance(child, QLineEdit):
                self.question_input = child
            elif obj_name == 'send' and isinstance(child, QPushButton):
                self.send_button = child

        try:
            # 위젯이 제대로 찾아졌는지 최종 확인
            if not self.answer_output or not self.question_input or not self.send_button:
                 # 어떤 위젯을 찾지 못했는지 사용자에게 알림
                 missing_widgets = []
                 if not self.answer_output: missing_widgets.append("answer (QTextEdit)")
                 if not self.question_input: missing_widgets.append("question (QLineEdit)")
                 if not self.send_button: missing_widgets.append("send (QPushButton)")
                 
                 raise ValueError(f"다음 위젯들을 찾을 수 없습니다: {', '.join(missing_widgets)}")

        except Exception as e:
            QMessageBox.critical(self, "UI 연결 오류", 
                                 f"위젯 연결 중 오류 발생: {e}. kang_gemini.ui 파일을 확인해주세요.")
            sys.exit(1)
        
        # 3. Gemini 클라이언트 및 채팅 세션 초기화
        self.client = self._initialize_gemini_client()
        self.chat = None
        if self.client:
            self.chat = self.client.chats.create(model="gemini-2.5-flash")
        
        # 4. 시그널 슬롯 연결
        self.send_button.clicked.connect(self.handle_send)

        # 5. 답변 위젯(QTextEdit)을 읽기 전용으로 설정합니다.
        self.answer_output.setReadOnly(True)

    def _initialize_gemini_client(self):
        """API 키로 Gemini 클라이언트를 초기화합니다."""
        try:
            return genai.Client(api_key=self.GEMINI_API_KEY)
        except Exception as e:
            QMessageBox.critical(self, "클라이언트 초기화 오류", f"Gemini 클라이언트 초기화 중 오류 발생: {e}")
            return None

    def handle_send(self):
        """질문을 처리하고 응답을 받아 출력하는 함수"""
        if not self.chat:
            QMessageBox.critical(self, "연결 오류", "Gemini 클라이언트가 초기화되지 않았습니다.")
            return

        user_prompt = self.question_input.text().strip()
        
        if not user_prompt:
            QMessageBox.warning(self, "경고", "질문을 입력해주세요.")
            return

        # 1. UI에 사용자 질문을 표시
        formatted_question = f"👤 [질문]\n{user_prompt}\n\n"
        self.answer_output.append(formatted_question)
            
        # 질문 입력창 초기화
        self.question_input.clear()
        
        # 2. Gemini API 호출
        try:
            response = self.chat.send_message(user_prompt)
            gemini_response = response.text
            
            # 3. UI에 Gemini 응답 표시
            formatted_answer = f"🤖 [kang_gemini]\n{gemini_response}\n\n"
            self.answer_output.append(formatted_answer)

        except APIError as e:
            QMessageBox.critical(self, "API 오류", f"Gemini API 호출 중 오류 발생: {e}")
        except Exception as e:
            QMessageBox.critical(self, "오류", f"예상치 못한 오류 발생: {e}")

if __name__ == '__main__':
    app = QApplication(sys.argv)
    window = GeminiChatApp()
    window.show()
    sys.exit(app.exec())
