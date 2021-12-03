FROM python:2.7-onbuild
ENV PYTHONPATH /usr/src/app
WORKDIR /app
ADD consumer.py /app
ADD requirements.txt /app
RUN python -m pip install -r /app/requirements.txt
CMD ["python /app/consumer.py", "-p 5672", "-s rabbitmq"]
