FROM python:2.7-onbuild
ENV PYTHONPATH /usr/src/app
WORKDIR /app
ADD ./consumer-app/consumer.py /app
ADD ./consumer-app/requirements.txt /app
ADD ./consumer-app/requirements.txt /usr/src/app
RUN apt update && apt install python3-pip && pip install --requirement /usr/src/app/requirements.txt
CMD ["python /app/consumer.py", "-p 5672", "-s rabbitmq"]
