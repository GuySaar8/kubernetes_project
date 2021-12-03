FROM python:2.7-onbuild
ENV PYTHONPATH /usr/src/app
WORKDIR /app
ADD consumer.py /app
ADD requirements.txt /app
RUN apt update && apt install python3-pip && pip install --requirement requirements.txt
CMD ["python /app/consumer.py", "-p 5672", "-s rabbitmq"]
