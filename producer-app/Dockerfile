FROM python:2.7-onbuild
ENV PYTHONPATH /usr/src/app
WORKDIR /app
ADD producer.py /app
ADD requirements.txt /app
RUN pip install --requirement /app/requirements.txt
CMD ["python producer.py", "-p 5672", "-s rabbitmq" "-m "Hello" -r 30"]
