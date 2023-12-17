FROM python:3.7-alpine
COPY app /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 8080

CMD ["python","app.py"]