---
layout: post
title: AOP Logging
---
I really like to use AOP because it increases modularity by allowing the separation of cross-cutting concerns. One such concern or commonality is using UnitOfWork at data access layer to manage database transactions which span multiple blocks of code. The below is implementation using only MethodInterceptor to achieve this 

The Annotation Class.


{% highlight java %}
import org.hibernate.CacheMode;
import org.hibernate.FlushMode;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.lang.annotation.Inherited;


@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface UnitOfWork {

    boolean readOnly() default false;

    boolean transactional() default true;

    CacheMode cacheMode() default CacheMode.NORMAL;

    FlushMode flushMode() default FlushMode.AUTO;

    boolean openSeparateTransaction() default false;
}
{% endhighlight %}

Method Interceptor Class

{% highlight java %}

import com.google.inject.Inject;
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.glassfish.jersey.server.internal.process.MappableException;
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.Transaction;
import org.hibernate.context.internal.ManagedSessionContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class UnitOfWorkInterceptor implements MethodInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(UnitOfWorkInterceptor.class);

    @Inject
    private SessionFactory sessionFactory;

    @Override
    public Object invoke(MethodInvocation arg0) throws Throwable {
        logger.debug("Annotation invoked on method {}", arg0.getMethod().getName());
        Session session = null;
        UnitOfWork unitOfWork = arg0.getMethod().getAnnotation(UnitOfWork.class);
        if (unitOfWork == null) {
            unitOfWork = arg0.getMethod().getDeclaringClass().getAnnotation(UnitOfWork.class);
        }
        if (!ManagedSessionContext.hasBind(sessionFactory) || unitOfWork.openSeparateTransaction()) {
            session = openSession(unitOfWork);
        } else {
            logger.debug("Session is already binded, not opening another session");
        }

        try {
            Object response = arg0.proceed();
            if (session != null) {
                closeSession(session, unitOfWork);
            }
            return response;
        } catch (Throwable t) {
            logger.error("Caught exception, throwing back, rolling back session {}", t.getMessage());
            if (session != null) {
                rollbackSession(session, unitOfWork);
            }
            throw t;
        }
    }

    private Session openSession(UnitOfWork unitOfWork) {
        logger.debug("Opening session with transactional as {}", unitOfWork.transactional());
        Session session = this.sessionFactory.openSession();
        try {
            configureSession(session, unitOfWork);
            ManagedSessionContext.bind(session);
            beginTransaction(session, unitOfWork);
        } catch (Throwable th) {
            closeSession(session);
            session = null;
            ManagedSessionContext.unbind(this.sessionFactory);
            throw th;
        }
        return session;
    }

    private void beginTransaction(Session session, UnitOfWork unitOfWork) {
        logger.debug("Beginning transaction with transactional as {}", unitOfWork.transactional());
        if (unitOfWork.transactional()) {
            session.beginTransaction();
        }
    }

    private void configureSession(Session session, UnitOfWork unitOfWork) {
        session.setDefaultReadOnly(unitOfWork.readOnly());
        session.setCacheMode(unitOfWork.cacheMode());
        session.setHibernateFlushMode(unitOfWork.flushMode());
    }

    private void closeSession(Session session, UnitOfWork unitOfWork) {
        logger.debug("Closing session with transactional as {}", unitOfWork.transactional());
        try {
            commitTransaction(session, unitOfWork);
        } catch (Exception e) {
            rollbackTransaction(session, unitOfWork);
            throw new MappableException(e);
        } finally {
            closeSession(session);
            session = null;
            ManagedSessionContext.unbind(this.sessionFactory);
        }
    }

    private void closeSession(Session session) {
        if (session != null && session.isOpen()) {
            logger.debug("Closing session");
            session.close();
        } else {
            logger.debug("Session is either null or already close, doing nothing");
        }
    }

    private void commitTransaction(Session session, UnitOfWork unitOfWork) {
        if (unitOfWork.transactional()) {
            final Transaction txn = session.getTransaction();
            if (txn != null && txn.isActive()) {
                logger.debug("Committing active transaction, active value {}", txn.isActive());
                txn.commit();
            }
        } else {
            logger.debug("Unit of work is not transactional, value = {}", unitOfWork.transactional());
        }
    }

    private void rollbackSession(Session session, UnitOfWork unitOfWork) {
        try {
            rollbackTransaction(session, unitOfWork);
        } finally {
            closeSession(session);
            session = null;
            ManagedSessionContext.unbind(this.sessionFactory);
        }
    }

    private void rollbackTransaction(Session session, UnitOfWork unitOfWork) {
        if (unitOfWork.transactional()) {
            final Transaction txn = session.getTransaction();
            if (txn != null && txn.isActive()) {
                logger.debug("Rolling back active transaction, active value {}", txn.isActive());
                txn.rollback();
            }
        }
    }

}
{% endhighlight %}
Sample Class

{% highlight java %}
public class DataService {

    private static final Logger logger = LoggerFactory.getLogger(DataService.class);

    private final DataDAO dataDAO;

    @Inject
    public DataService(DataDAO dataDAO) {
        super(dataDAO);
    }

    @UnitOfWork
    public void createData(DataRequest dataRequest) {
        DataDO dataDO = transform(dataRequest);
        dataDao.save(dataDO);
    }
}
{% endhighlight %}

Injection using Guice.

{% highlight java %}
@Override
protected void configure() {
    MethodInterceptor unitOfWorkInterceptor = new UnitOfWorkInterceptor();
    requestInjection(unitOfWorkInterceptor);
    bindInterceptor(any(), annotatedWith(UnitOfWork.class), unitOfWorkInterceptor);
    bindInterceptor(annotatedWith(UnitOfWork.class), any(), unitOfWorkInterceptor);
}
{% endhighlight %}
